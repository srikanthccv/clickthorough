In one of his [presentations](https://presentations.clickhouse.com/cern/#40), Alexey Milovidov, the author of ClickHouse, talks about what makes it so fast. The **_we have 17 different algorithms for GROUP BY. Best one is selected for your query_** part caught my attention. I decided to dig deeper and find out what are these 17 algorithms. The presentation is about 7 years old now. Since then, several more optimizations for GROUP BY have been added. At the time of this writing, there are around 36+ optimizations for GROUP BY aggregations. In this post, we will look at 36+ optimizations ClickHouse uses for GROUP BY aggregations.

There are a number of datasets available to work with. Let's load the _New York Taxi Data_ dataset and play with some queries. You can find them [here](https://clickhouse.com/docs/en/getting-started/example-datasets). We are interested in seeing everything ClickHouse does, so let's change the `send_logs_level=trace` setting to capture all the logs. 

In the `clickhouse-client`, you can run the following command to change the `send_logs_level` setting to `trace`:

```
SET send_logs_level = 'trace'
```

Now, Let's run the query to show the top 10 neighborhoods that have the most frequent pickups.

```sql
SELECT
    pickup_ntaname,
    count(*) AS count
FROM trips
GROUP BY pickup_ntaname
ORDER BY count DESC
LIMIT 10
```

The following line shows up in the logs:

```
[Srikanths-MacBook-Pro.local] 2024.04.13 12:22:07.768550 [ 1171191 ] {d3cace38-0f4f-4f02-bce6-142ed60501d2} <Trace> Aggregator: Aggregation method: low_cardinality_key_string
```

Hmm, interesting. It shows that ClickHouse has chosen the `low_cardinality_key_string` method for the query. What is this method? How does it decide what method to use? How does it affect the performance? So many questions. Come on in fella, we are going to find out.

At the heart of ClickHouse's GROUP BY algorithm is the [`chooseAggregationMethod`](https://github.com/ClickHouse/ClickHouse/blob/83c050ad3606521f977ca930e3f33d3347b5b199/src/Interpreters/Aggregator.cpp#L621-L832) function. The function selects the most efficient method for aggregating data. It chooses the most appropriate method based on the number and data type of keys.

Let's break down the function. 

```c++
    /// If no keys. All aggregating to single row.
    if (params.keys_size == 0)
        return AggregatedDataVariants::Type::without_key;
```

This is the simplest case. If the query does not have a key, it aggregates all the data to a single row.

```c++
    /// Check if at least one of the specified keys is nullable.
    DataTypes types_removed_nullable;
    types_removed_nullable.reserve(params.keys.size());
    bool has_nullable_key = false;
    bool has_low_cardinality = false;

    for (const auto & key : params.keys)
    {
        DataTypePtr type = header.getByName(key).type;

        if (type->lowCardinality())
        {
            has_low_cardinality = true;
            type = removeLowCardinality(type);
        }

        if (type->isNullable())
        {
            has_nullable_key = true;
            type = removeNullable(type);
        }

        types_removed_nullable.push_back(type);
    }
```

This piece of code checks if there are any nullable or low cardinality keys. This is the first hint that ClickHouse uses key's properties to decide what method to use.

Immediately follows the interesting code comment; Let's talk a bit about the two-level thing. 

```
    /** Returns ordinary (not two-level) methods, because we start from them.
      * Later, during aggregation process, data may be converted (partitioned) to two-level structure, if cardinality is high.
      */
```


When you run the query, you will see the following lines in the logs:

```
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.588280 [ 1171191 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 368640 to 172 rows (from 10.59 MiB) in 0.4114915 sec. (895862.977 rows/sec., 25.74 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.588598 [ 1171192 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 368640 to 172 rows (from 10.63 MiB) in 0.41158325 sec. (895663.271 rows/sec., 25.83 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.593291 [ 1181655 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 362092 to 170 rows (from 10.44 MiB) in 0.417350792 sec. (867596.293 rows/sec., 25.01 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.593991 [ 1171194 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 352256 to 168 rows (from 10.09 MiB) in 0.417722625 sec. (843277.282 rows/sec., 24.16 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.594637 [ 1179291 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 376832 to 173 rows (from 10.81 MiB) in 0.418044459 sec. (901416.086 rows/sec., 25.85 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.595626 [ 1171186 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 368640 to 172 rows (from 10.64 MiB) in 0.419871917 sec. (877982.035 rows/sec., 25.35 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.597017 [ 1171193 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 360448 to 170 rows (from 10.36 MiB) in 0.420922416 sec. (856328.830 rows/sec., 24.62 MiB/sec.)
[Srikanths-MacBook-Pro.local] 2024.04.13 20:15:42.621890 [ 1171187 ] {d1b3cb6e-cbc6-488b-bcf8-1d0eb69b1206} <Trace> AggregatingTransform: Aggregated. 442769 to 179 rows (from 12.77 MiB) in 0.445446167 sec. (993989.920 rows/sec., 28.66 MiB/sec.)
```

As you might have guessed, ClickHouse launched as many threads as the number of CPU cores. Each thread independently aggregates the part of the data. Now, the intermediary results should be combined to get the final result. This step of combining the results is done sequentially, which is fine for small data sets. But for large data sets, this might be slow. To overcome this, ClickHouse uses the two-level structure. The idea is each thread introduces 256 small hash tables at the first level. The intermediary results combination process is parallelized by bucket number. However, this comes with the cost of additional memory usage. If the group by key doesn't have high unique keys, it's a waste of memory to allocate some thousands of these small hash tables.


```c++
    size_t keys_bytes = 0;
    size_t num_fixed_contiguous_keys = 0;

    key_sizes.resize(params.keys_size);
    for (size_t j = 0; j < params.keys_size; ++j)
    {
        if (types_removed_nullable[j]->isValueUnambiguouslyRepresentedInContiguousMemoryRegion())
        {
            if (types_removed_nullable[j]->isValueUnambiguouslyRepresentedInFixedSizeContiguousMemoryRegion())
            {
                ++num_fixed_contiguous_keys;
                key_sizes[j] = types_removed_nullable[j]->getSizeOfValueInMemory();
                keys_bytes += key_sizes[j];
            }
        }
    }

    bool all_keys_are_numbers_or_strings = true;
    for (size_t j = 0; j < params.keys_size; ++j)
    {
        if (!types_removed_nullable[j]->isValueRepresentedByNumber() && !isString(types_removed_nullable[j])
            && !isFixedString(types_removed_nullable[j]))
        {
            all_keys_are_numbers_or_strings = false;
            break;
        }
    }
```

TODO: Explain

What follows next is a big if/else statement. 


<details open>

<summary>Code</summary>

```c++
    if (has_nullable_key)
    {
        /// Optimization for one key
        if (params.keys_size == 1 && !has_low_cardinality)
        {
            if (types_removed_nullable[0]->isValueRepresentedByNumber())
            {
                size_t size_of_field = types_removed_nullable[0]->getSizeOfValueInMemory();
                if (size_of_field == 1)
                    return AggregatedDataVariants::Type::nullable_key8;
                if (size_of_field == 2)
                    return AggregatedDataVariants::Type::nullable_key16;
                if (size_of_field == 4)
                    return AggregatedDataVariants::Type::nullable_key32;
                if (size_of_field == 8)
                    return AggregatedDataVariants::Type::nullable_key64;
            }
            if (isFixedString(types_removed_nullable[0]))
            {
                return AggregatedDataVariants::Type::nullable_key_fixed_string;
            }
            if (isString(types_removed_nullable[0]))
            {
                return AggregatedDataVariants::Type::nullable_key_string;
            }
        }

        if (params.keys_size == num_fixed_contiguous_keys && !has_low_cardinality)
        {
            /// Pack if possible all the keys along with information about which key values are nulls
            /// into a fixed 16- or 32-byte blob.
            if (std::tuple_size<KeysNullMap<UInt128>>::value + keys_bytes <= 16)
                return AggregatedDataVariants::Type::nullable_keys128;
            if (std::tuple_size<KeysNullMap<UInt256>>::value + keys_bytes <= 32)
                return AggregatedDataVariants::Type::nullable_keys256;
        }

        if (has_low_cardinality && params.keys_size == 1)
        {
            if (types_removed_nullable[0]->isValueRepresentedByNumber())
            {
                size_t size_of_field = types_removed_nullable[0]->getSizeOfValueInMemory();

                if (size_of_field == 1)
                    return AggregatedDataVariants::Type::low_cardinality_key8;
                if (size_of_field == 2)
                    return AggregatedDataVariants::Type::low_cardinality_key16;
                if (size_of_field == 4)
                    return AggregatedDataVariants::Type::low_cardinality_key32;
                if (size_of_field == 8)
                    return AggregatedDataVariants::Type::low_cardinality_key64;
            }
            else if (isString(types_removed_nullable[0]))
                return AggregatedDataVariants::Type::low_cardinality_key_string;
            else if (isFixedString(types_removed_nullable[0]))
                return AggregatedDataVariants::Type::low_cardinality_key_fixed_string;
        }

        if (params.keys_size > 1 && all_keys_are_numbers_or_strings)
            return AggregatedDataVariants::Type::nullable_prealloc_serialized;

        /// Fallback case.
        return AggregatedDataVariants::Type::nullable_serialized;
    }

    /// No key has been found to be nullable.

    /// Single numeric key.
    if (params.keys_size == 1 && types_removed_nullable[0]->isValueRepresentedByNumber())
    {
        size_t size_of_field = types_removed_nullable[0]->getSizeOfValueInMemory();

        if (has_low_cardinality)
        {
            if (size_of_field == 1)
                return AggregatedDataVariants::Type::low_cardinality_key8;
            if (size_of_field == 2)
                return AggregatedDataVariants::Type::low_cardinality_key16;
            if (size_of_field == 4)
                return AggregatedDataVariants::Type::low_cardinality_key32;
            if (size_of_field == 8)
                return AggregatedDataVariants::Type::low_cardinality_key64;
            if (size_of_field == 16)
                return AggregatedDataVariants::Type::low_cardinality_keys128;
            if (size_of_field == 32)
                return AggregatedDataVariants::Type::low_cardinality_keys256;
            throw Exception(ErrorCodes::LOGICAL_ERROR, "LowCardinality numeric column has sizeOfField not in 1, 2, 4, 8, 16, 32.");
        }

        if (size_of_field == 1)
            return AggregatedDataVariants::Type::key8;
        if (size_of_field == 2)
            return AggregatedDataVariants::Type::key16;
        if (size_of_field == 4)
            return AggregatedDataVariants::Type::key32;
        if (size_of_field == 8)
            return AggregatedDataVariants::Type::key64;
        if (size_of_field == 16)
            return AggregatedDataVariants::Type::keys128;
        if (size_of_field == 32)
            return AggregatedDataVariants::Type::keys256;
        throw Exception(ErrorCodes::LOGICAL_ERROR, "Numeric column has sizeOfField not in 1, 2, 4, 8, 16, 32.");
    }

    if (params.keys_size == 1 && isFixedString(types_removed_nullable[0]))
    {
        if (has_low_cardinality)
            return AggregatedDataVariants::Type::low_cardinality_key_fixed_string;
        else
            return AggregatedDataVariants::Type::key_fixed_string;
    }

    /// If all keys fits in N bits, will use hash table with all keys packed (placed contiguously) to single N-bit key.
    if (params.keys_size == num_fixed_contiguous_keys)
    {
        if (has_low_cardinality)
        {
            if (keys_bytes <= 16)
                return AggregatedDataVariants::Type::low_cardinality_keys128;
            if (keys_bytes <= 32)
                return AggregatedDataVariants::Type::low_cardinality_keys256;
        }

        if (keys_bytes <= 2)
            return AggregatedDataVariants::Type::keys16;
        if (keys_bytes <= 4)
            return AggregatedDataVariants::Type::keys32;
        if (keys_bytes <= 8)
            return AggregatedDataVariants::Type::keys64;
        if (keys_bytes <= 16)
            return AggregatedDataVariants::Type::keys128;
        if (keys_bytes <= 32)
            return AggregatedDataVariants::Type::keys256;
    }

    /// If single string key - will use hash table with references to it. Strings itself are stored separately in Arena.
    if (params.keys_size == 1 && isString(types_removed_nullable[0]))
    {
        if (has_low_cardinality)
            return AggregatedDataVariants::Type::low_cardinality_key_string;
        else
            return AggregatedDataVariants::Type::key_string;
    }

    if (params.keys_size > 1 && all_keys_are_numbers_or_strings)
        return AggregatedDataVariants::Type::prealloc_serialized;

    return AggregatedDataVariants::Type::serialized;
```

</details>

You can notice mainly two things in the code. First, when the number of keys = 1, the aggregation method is determined by the size of the key. Second, when the number of keys > 1, the aggregation method is determined by the total size of the keys and if they can be packed into a single N-bit key. Lastly, the fallback case is a serialized method. The **_serialized_** method is the slowest among all the methods. So when you write queries, you should keep in mind this detail and if needed, try to re-write the query so that ClickHouse can choose the most efficient method.

### Examples


1. Column with the same data, different data types

Let's create a new column `pickup_ntaname_string` as a string type but with the same data as `pickup_ntaname`.

```
ALTER TABLE datasets.trips ADD COLUMN pickup_ntaname_string String MATERIALIZED toString(pickup_ntaname);
ALTER TABLE datasets.trips MATERIALIZE COLUMN pickup_ntaname_string;
```

Let's run the query again using the new column.

```sql
SELECT
    pickup_ntaname_string,
    count(*) AS count
FROM trips
GROUP BY pickup_ntaname_string
ORDER BY count DESC
LIMIT 10
```

Even though the data set is the same, the query is at least twice slower because the selected aggregation method (**_key\_string_**) is different, which means the hash table used is different. 

2. Two low cardinality keys

```sql
SELECT
    pickup_ntaname,
    dropoff_ntaname,
    count(*) AS count
FROM trips
GROUP BY pickup_ntaname, dropoff_ntaname
ORDER BY count DESC
LIMIT 10
```

Since there is more than one key, ClickHouse checks if the keys can be packed into a single N-bit key. If they can, ClickHouse will use the **_keys{N}** method. If not, ClickHouse will use the **_prealloc_serialized_** or **_serialized_** method based on the data type of all the keys.

The following query truncates the strings to 16 characters and then groups by the truncated strings to show that the selected method will be **_keys256_**.

```
SELECT
    CAST(left(pickup_ntaname, 16), 'FixedString(16)') as pname,
    CAST(left(dropoff_ntaname, 16), 'FixedString(16)') as dname,
    count(*) AS count
FROM trips
GROUP BY
    pname,
    dname
ORDER BY count DESC
LIMIT 10
```