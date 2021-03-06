# In Depth Guide
This is a start guide for how the internals of CfGRIB.jl work, targeted
towards advanced users or those who want to work with the internals of the code.

If you want a quick guide on how to use the package then check the [`Quick Start
Guide`](@ref)

## Internals
The package internals are covered in the library section of the documentation in
greater detail, however it is useful to have a vague sense of what is happening
when you load a dataset.

First, we load the package, and for convenience create a string pointing to our
file path:

```@repl 1
using CfGRIB

sample_data_dir = abspath(joinpath(dirname(pathof(CfGRIB)), "..", "test", "sample-data"))

demo_file_path = joinpath(sample_data_dir, "era5-levels-members.grib")
```

### `FileIndex`
Whenever you load a `grib` file, the first thing that happens is that the file
index is read. The file index contains metadata which describes which messages
contain what information inside the file. We can explore the index by manually
creating a [`FileIndex`](@ref FileIndex) object.

First, we can look at the docstring for the `FileIndex` constructor by typing in
`?` at the REPL to enter `help` mode, then type in `CfGRIB.FileIndex`, press
enter, and we get the docstring:

```@example 1
help?> CfGRIB.FileIndex
  Summary
  ≡≡≡≡≡≡≡≡≡

  mutable struct FileIndex

  A mutable store for indices of a GRIB file

  TODO: Should probably change this to a immutable struct

  Fields
  ≡≡≡≡≡≡≡≡

    •    allowed_protocol_version::VersionNumber

        Version number used when saving/hashing index files, should change if
        the indexing structure changes breaking backwards-compatibility

    •    grib_path::String

        Path to the file the index belongs to

    •    index_path::String

        Path to the index cache file

    •    index_keys::Array{String,1}

        Array containing all of the index keys

    •    offsets::Array{Pair{NamedTuple,Int64},1}

        Array containing pairs of offsets[HeaderTuple(header_values)] => offset_field

    •    message_lengths::Array{Int64,1}

        Array containing the length of each message in the GRIB file

    •    header_values::OrderedCollections.OrderedDict{String,Array}

        Dictionary of all of the loaded header values in the GRIB file

    •    filter_by_keys::Dict

        Filters used when creating the file index

  Constructors
  ≡≡≡≡≡≡≡≡≡≡≡≡≡≡

  FileIndex()

  defined at dev/CfGRIB/src/indexing.jl:34
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/indexing.jl#L34).

  FileIndex(grib_path, index_keys)

  defined at dev/CfGRIB/src/indexing.jl:38
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/indexing.jl#L38).
```

The docstring is quite long, it explains the fields contained in the object, as
well as giving a list of the constructors which can be used to create an
instance of the object.

We'll use the second constructor, which takes in a path to the file and a list
of keys. First, we pick which keys we want to use. In this case we'll just use
the `ALL_KEYS` constant:

```@repl 1
println(CfGRIB.ALL_KEYS)

index = CfGRIB.FileIndex(
           demo_file_path,
           CfGRIB.ALL_KEYS
       );
```

From here you can explore fields contained in this object. Typically you will
never interact with the `FileIndex` directly, as it's just used in the
background to load the data.

### `DataSet`
Once the `FileIndex` has been created, the next step is to use it to create a
[`DataSet`](@ref DataSet) object. The `DataSet` is what what you use to access
the stored data. The docstring says:

```@example 1
help?> CfGRIB.DataSet
  Summary
  ≡≡≡≡≡≡≡≡≡

  struct DataSet

  Map a GRIB file to the NetCDF Common Data Model with CF Conventions.

  Fields
  ≡≡≡≡≡≡≡≡

    •    dimensions::OrderedCollections.OrderedDict{String,Int64}

        OrderedDict{String,Int} of $DIMENSION_NAME => $DIMENSION_LENGTH.

    •    variables::OrderedCollections.OrderedDict{String,CfGRIB.Variable}

        OrderedDict{String,CfGRIB.Variable} of $DIMENSION_NAME => $DIMENSION_VARIABLE, where the the variable is a CfGRIB.jl Variable.

    •    attributes::OrderedCollections.OrderedDict{String,Any}

        OrderedDict{String,Any} containing some metadata extracted from the file.

    •    encoding::Dict{String,Any}

        Dict{String,Any} containing metadata related to CfGRIB.jl, e.g. filter_by_keys

  Constructors
  ≡≡≡≡≡≡≡≡≡≡≡≡≡≡

  DataSet(dimensions, variables, attributes, encoding)

  defined at dev/CfGRIB/src/dataset.jl:127
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L127).

  DataSet(path; read_keys, kwargs...)

  defined at dev/CfGRIB/src/dataset.jl:140
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L140).
```

Here we see references to [`Variable`](@ref Variable), so we'll briefly explain
those.

#### `Variable`
A `Variable` is a basic struct in CfGRIB.jl which contains information for a
variable read from a GRIB file:

```julia
help?> CfGRIB.Variable
  Summary
  ≡≡≡≡≡≡≡≡≡

  struct Variable

  Struct describing a cfgrib variable

  Fields
  ≡≡≡≡≡≡≡≡

    •    dimensions::Tuple{Vararg{String,N} where N}

        Name of the dimension(s) contained in this variable

    •    data::Union{CfGRIB.OnDiskArray, Number, Array}

        Data contained in the variable, can point ot in-memory data or to a CfGRIB
        OnDiskArray

    •    attributes::Dict{String,Any}

        Dictionary containing metadata for the variable, typically the units, the long name,
        and the standard name

  Constructors
  ≡≡≡≡≡≡≡≡≡≡≡≡≡≡

  Variable(dimensions, data, attributes)

  defined at dev/CfGRIB/src/dataset.jl:108
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L108).
```

#### `OnDiskArray`

As explained above, `Variable`s contain a `data` field, this data can either be
in-memory data (`Array`, `Number`), or it could be an `OnDiskArray`. On disk
arrays are, as the name hints, a way to represent data stored on the disk
before that data is loaded.

This is done do make it a bit easier to deal with large datasets, as the data is
only lazily loaded in when the user attempts to read it. And then, only the
requested data is stored in memory.

```julia
help?> CfGRIB.OnDiskArray
  Summary
  ≡≡≡≡≡≡≡≡≡

  struct OnDiskArray

  Struct that contains metadata for an array, used to lazy-load the array from disk only when
  requested

  Fields
  ≡≡≡≡≡≡≡≡

    •    grib_path::String

    •    size::Tuple

    •    offsets::OrderedCollections.OrderedDict

    •    message_lengths::Array{Int64,1}

    •    missing_value::Any

    •    geo_ndim::Int64

    •    dtype::Type

  Constructors
  ≡≡≡≡≡≡≡≡≡≡≡≡≡≡

  OnDiskArray(grib_path, size, offsets, message_lengths, missing_value, geo_ndim, dtype)

  defined at dev/CfGRIB/src/dataset.jl:27
  (https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L27).
```

The `OnDiskArray` object contains enough information to fully describe the data
stored on disk, and to allow for easy indexing into this data. A custom
`getindex` method dispatches off of this type which opens the grib file at
`grib_path` and reads only the relevant messages.

For example, if a 3 dimensional array is described by `OnDiskArray`, and the
user requests information with index `[1, :, :]`, then only messages within
that index are loaded from the grib file.

### `DataSet` Constructors

Now that the groundwork is laid down, lets look into how files are read and used
in the end. The most basic option is calling `DataSet` with a string as a path,
this will use the constructor defined at dev/CfGRIB/src/dataset.jl:140
(https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L140).

As you can see this creates a `FileIndex`, and then returns:

```julia
DataSet(build_dataset_components(
    index;
    errors=errors,
    encode_cf=encode_cf,
    squeeze=squeeze,
    read_keys=read_keys,
    time_dims=time_dims,
)...)
```

The call to [`build_dataset_components`](@ref) returns the dimensions,
variables, attributes, and encoding read from a file. These four variables are
then passed to the other relevant constructor defined at
dev/CfGRIB/src/dataset.jl:127
(https://github.com/ecmwf/cfgrib.jl/tree/5ced129d540ed9a1ff57da48c9b4f047b17d936d//src/dataset.jl#L127).

The constructor then returns a `DataSet` object.

## Getting Data from a `DataSet`

Onc you have a `DataSet` object, you probably want to access its data.

### Direct Access

The most basic way to do this is to just access the `variables` directly. For
example:

```@repl 1
dataset = CfGRIB.DataSet(demo_file_path);

dataset.dimensions

dataset.variables
```

From here you can check the [`Variable`](@ref) documentation to see what is
stored in these. So, if we want to get the data for `z`:

```@repl 1
dataset.variables["z"]

dataset.variables["z"].data

convert(Array, dataset.variables["z"].data)[:, :, 1, 1, 1]
```

Since it's an [`OnDiskArray`](@ref) it has to be `converted` (which in this case
just reads the data from disk) into an Array. Once that's done, it's just a
standard array type which can be accessed.

For a normal variable stored in memory this is a bit easier as the reading step
does not have to be performed:

```@repl 1
dataset.variables["number"]

dataset.variables["number"].data
```

Accessing all of the data this way would be extremely awkward, so we provide a
number of multidimensional named-axis backends which make data access far
easier.

### Using Named Dimensional Backends

The recommended way to use CfGRIB.jl is to use an array backend. More
information about backends can be found on the [Backends](@ref lib_backends)
documentation page.

If one of the backend dependencies is available you can convert to that backend
data type with the `convert` function:

```@repl 1
using AxisArrays

dimensional_dataset = convert(AxisArray, dataset)
```

This conversion to a backend will create an object for that specific backend,
preserving all of the data that was present in our `DataSet` objects (e.g. the
metadata will all be propagated through).

Current backend implementations have two limitations:
1. No 'dataset' like support
2. No metadata support

These limitations mean that we have to create a wrapper struct which can hold
the multidimensional array type from the backend, as well as some additional
attributes.

In the python `xarray` package, there are two basic types: a `DataArray` and a
`DataSet`. The `DataArray` is a multidimensional array of a single variable,
which contains information for that variable as well as information about the
dimensions which enables useful indexing capabilities.

The `DataSet` is a set of **multiple** `DataArray`s with common dimensions. This
lets you have a `DataArray` containing pressure information with dimensions of,
for example, time, latitude, longitude, and height; if you have another set of
data with the same dimensions but for temperature then you can store both in a
singe `DataSet`.

The backends we currently use do not have this functionality, so instead we just
wrap the two variables and allow for easy access to both.

Additionally, our `DataSet` contains some more metadata (such as the attributes
and encoding information), which also cannot be stored in the array backends,
so we store that in the wrapper as well.

To access the data you first access a single specific dataset and then index
into it as per the docs for your chosen backend. For example, above we use
`AxisArrays` as the backend, so:

```@repl 1
using AxisArrays

z = dimensional_dataset.z;  # Looking at the `z` variable

z[number=atvalue(0), isobaricInhPa=700..900, longitude=40..44]
```

