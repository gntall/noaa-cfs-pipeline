# User Requirements

1. from DS team 
    1. ********************data lake******************** entire archive once a month, must take < 3rhs to transfer to a different machine
    2. **processed data** slices of processed data of 7 days worths of predictions (with all lead times) 
2. back end team
    1. **processed data** one prediction with all lead times of 1000 locations within slice of longitudes, JSON format, every 1 hr, 2 minute time limit
    2. **processed data** one prediction with all lead times of 1 location, JSON format, 100 times per hour, 2-second time limit

# Ingest layer

- Overview
    1. use webhook to find when the latest data is published to URL
    2. Lambda compute puts chunks into stream
    3. Stream passes batches to raw staging area for cleaning
    4.  Cluster cleans raw data and pushes to raw ****************************data lake****************************
- **API gateway** wakes Lambda to initiate chunk processing
    - Get base url, need to make sure we don’t call each link many times because of the time out limitation and no simultaneous IP calls
- **Lambda** pushes individual `GRIB` files into stream
    - Lambda code coordinates the API calls according to the API call constraints like number of concurrent calls, timeout between each call
    - Lambda can have memory up to 10GB, so able to process more than one chunk in parallel
    - Using a **single Lambda** can be a potential bottleneck, but works well for putting the binary chunks into a stream fast. For higher throughput, we could add more lambda functions. These will have separate IP addresses.
- **Kinesis** dumps chunks into raw staging area
    - Stream allows for increased throughput and rapid checks before putting in staging area
    - we don’t need it up and running for the remaining 5 hour intervals, so Kinesis should be in **on-demand mode** rather than provision mode
    - check for data corruption/transferred incorrectly with check sums.
        - If the checksum fails, can reload chunks polled from Lambda with retry policy if needed
- Send run logs to **********Cloudwatch********** and error/retry notifications
    - run logs
    - config and metadata

## Raw data processing

- ********************Container******************** performs data validation on `grib` data in staging area
    - **data sanity** checks
        - data types match, dimensions are correct for each model
        - if sanity check fails, delete run and notify through notification service
    - **data quality** checks
        - correct timestamps, grid boundaries
        - if quality checks fail, send notification and have configurale option of whether to roll back or not. Recommended route is acceptance scale rather than pass/fail. When possible, use domain knowledge to inform tolerance range of “dirty” raw data. This allows to move ahead and allow for some imperfection in measurements. This can be stored in a validation config
    - for scaling up preprocessing, containers can be part of Fargate cluster with autoscaling
- ******************Container****************** transforms data and save to **********************raw data lake**********************
    - transform grib files to zarr and filter for correct variables from run config
    - save `.zarr` file and store with data lake hierarchy to file storage.
- Optimizations
    - perform transform operations on chunks into minibatches by container. parallelize the cleaning of the data, since launching a container to clean each chunk would take more time
    - make containers part of Fargate cluster with auto-scale to handle the data format transformations
- Send notification once process is complete
- Once transfer from stage to raw data lake is complete, delete original files or send to cold storage (AWS Glacier flexible retrieval)
    - ** AWS Glacier coldest storage would be a ~600$ a month for permanent storage of full raw grb2 data. Not requested but here’s the quote!

## Data lake storage

- **********zarr********** is the format of choice for the data lake. The data is partitioned with zarr with hierarchies, and chunk sizes. Example hierarchy for raw data lake will be `model/version/init_timest/lead_time/file.zarr`
- Once all files are ready, trigger the processing compute (this can be EMR)
- Use customized storage classes for latest week of data for faster access, and use colder lass for older/archival data. S3 Infrequent access or Amazon S3 Glacier Instant Retrieval. (this addresses reqs 1a, 1b)

### zarr vs parquet

Zarr provides the following benefits

- optimized for chunk-based distributed retrieval. This makes it good for retrieving chunks of data along similar longitude ranges
- it provides flexibility when dealing with models of different levels (not just coordinates, also depth, height) using hierarchical groups
- customize chunk size and dimensionality. For example, I can choose to chunk across one dimension and not the other

Even though **parquet** provides similar compression beneifts as zarr, it provides the following downsides given our requirements:

- it require custom setup for models with different levels, making it less flexible in the long run
- it has slow random lookup, so would make individual location requests slower than zarr, and not satisfy req 2b

The system requires fast random access (req 2b), and retrieval along chunks of coordinates (2a), therefore it is the preferred choice of storage format for the processed data storage.

See [here](https://www.azavea.com/blog/2022/09/22/benchmarking-zarr-and-parquet-data-retrieval-using-the-national-water-model-nwm-in-a-cloud-native-environment/) for benchmark example on weather data

### Partitioning zarr

**Raw data lake hierarchy** 

- partition by model, model version, init time, lead time
- store long and narrow chunks for optimal bluk transfer

**Processed data hierarchy** 

- partition model, init time, lead time, coordinate ranges
- store less long chunks for optimal random access and flexibility in slicing and aggregating

# Processing

- ******************Container****************** gets data ready for ML
    - Regridding process uses EMR Spark cluster to optimize to parallelize the preprocessing as much as possible
- ML orchestration and MLOps
    - Sagemaker is valid MLOps tool which provides ML model registry, data versioning, and metal for model runs in single package
    - otherwise can use manages K8s cluster for model running paired with MLFlow for model registry and versioning
- Each ML container requires access to code packages for the following features
    - pipeline objects
    - model run configurations
    - access to common config codes
    - log/monitoring app connector
    - output summary statistics to run logs
    - once process is done, launch ML model output checker container
- ML output check container
    - access model output summary statistics

# Serving layer

- ************************API************************ plus light compute instance for retrieving data quickly
    - Lambda can run distributed access packages such as `fsspec` and  `xarray` to slice the `zarr` data accordingly
    - transform output into
- Transfer jobs for bulk archive transfer
    - Schedule bulk transfers with cloud provider
    - Blob storage custom with customized caching policies to keep

## Common code interfaces

- Run configs
    - save raw run config files for model, columns information, validation parameters
- Handle grib files
    - open with xarray
    - convert to zarr
- Datalake connector
    - access to zarr for custom slicing and aggregation
    - use packages like fsspec, xarray, dask
- Check ETL run logs
    - check summary statistics
- Validation
    - Checksums
    - Sanity checks (data types match, dimensions match for each model)
    - Quality checks (summary statistics, value ranges)
