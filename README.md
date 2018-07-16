# BEDFiles.jl
Routines for reading and manipulating GWAS data in .bed files

| **Documentation**                                                               | **PackageEvaluator**                                            | **Build Status**                                                                                |
|:-------------------------------------------------------------------------------:|:---------------------------------------------------------------:|:-----------------------------------------------------------------------------------------------:|
| [![][docs-stable-img]][docs-stable-url] [![][docs-latest-img]][docs-latest-url] | [![][pkg-0.7-img]][pkg-0.7-url] | [![][travis-img]][travis-url] [![][appveyor-img]][appveyor-url] [![][coveralls-img]][coveralls-url] |

Data from [*Genome-wide association studies*](https://en.wikipedia.org/wiki/Genome-wide_association_study)
are often saved as a [**PLINK binary biallelic genotype table**](https://www.cog-genomics.org/plink2/formats#bed)
or `.bed` file.
To be useful, such files should be accompanied by a `.fam` file, containing metadata on the rows of the table, and a `.bim` file,
containing metadata on the columns.
The `.fam` and `.bim` files are in tab-separated format.

The table contains the observed allelic type at `n`
[*single-nucleotide polymorphism*](https://en.wikipedia.org/wiki/Single-nucleotide_polymorphism) (SNP) positions 
for `m` individuals.

A SNP corresponds to a nucleotide position on the genome where some degree of variation has been observed in a population,
with each individual have one of two possible *alleles* at that position on each of a pair of chromosomes.
Three possible types can be observed are:
homozygous allele 1, coded as `0x00`, heterzygous, coded as `0x10`, and homozygous allele 2, coded as `0x11`.
Missing values are coded as `0x01`.

A single column - one SNP position over all `m` individuals - is packed into an
array of `div(m + 3, 4)` bytes (`UInt8` values).

## Installation

This package requires Julia v0.7.0-beta or later, which can be obtained from
https://julialang.org/downloads/ or by building Julia from the sources in the
https://github.com/JuliaLang/julia repository.

The package has not yet been registered and must be installed using the repository location.
Start julia and use the `]` key to switch to the package manager REPL
```julia
(v0.7) pkg> add https://github.com/dmbates/BEDFiles.jl.git#master
  Updating git-repo `https://github.com/dmbates/BEDFiles.jl.git`
  Updating registry at `~/.julia/registries/Uncurated`
  Updating git-repo `https://github.com/JuliaRegistries/Uncurated.git`
 Resolving package versions...
  Updating `~/.julia/environments/v0.7/Project.toml`
  [6f44c9a6] + BEDFiles v0.1.0 #master (https://github.com/dmbates/BEDFiles.jl.git)
  Updating `~/.julia/environments/v0.7/Manifest.toml`
  [6f44c9a6] + BEDFiles v0.1.0 #master (https://github.com/dmbates/BEDFiles.jl.git)
  [6fe1bfb0] + OffsetArrays v0.6.0
  [10745b16] + Statistics 
```

Use the backspace key to return to the Julia REPL.

## Loading a .bed file

The `BEDFile` struct contains the read-only, memory-mapped `.bed` file as a `Matrix{UInt8}`,
along with `m`, the number of individuals.
The columns correspond to SNP positions.
Rows of the internal matrix are packed values from groups of 4 individuals.

```julia
julia> using Pkg

julia> using BenchmarkTools, BEDFiles

julia> const bf = BEDFile(Pkg.dir("BEDFiles", "data", "mouse", "alldata.bed")
BEDFile(UInt8[0xba 0xba … 0xff 0xff; 0xab 0xab … 0xfe 0xfe; … ; 0xbb 0xbb … 0xff 0xff; 0x2a 0x2a … 0xdf 0xdf], 1940)

julia> size(bf)      # the virtual size of the GWAS data - 1940 observations at each of 10150 SNP positions
(1940, 10150)

julia> size(bf.data) # the actual size of the memory-mapped matrix of UInt8s
(485, 10150)
```

As described above, a column, consisting of `m` values in the range `0x00` to `0x03`, is packed into `div(m + 3, 4)` bytes.
```julia
julia> div(1940 + 3, 4)   # an equivalent, and somewhat faster, calculation is ((1940 + 3) >> 2)
485
```

The virtual number of rows, `m`, can be given as a second argument in the call to `BEDFile`.
If omitted, `m` is determined as the number of lines in the `.fam` file. 

Because the file is memory-mapped this operation is fast, even for very large `.bed` files.
```julia
julia> @time BEDFile("./data/mouse/alldata.bed");
  0.000316 seconds (48 allocations: 10.500 KiB)
```

This file, from a study published in 2006, is about 5 Mb in size but data from recent studies, which have samples from tens of
thousands of individuals at over a million SNP positions, would be in the tens or even hundreds of Gb range.
## Raw summaries

Counts of each the four possible values for each column are returned by `columncounts`.

```julia
julia> columncounts(bf)
4×10150 Array{Int64,2}:
  358   359  252   358    33   359    33  186   360  …    53    56    56    56    56    56    56    56
    2     0    4     3     4     1     4    1     3      171   174   173   173   162   173   174   175
 1003  1004  888  1004   442  1004   481  803  1002      186   242   242   242   242   242   242   242
  577   577  796   575  1461   576  1422  950   575     1530  1468  1469  1469  1480  1469  1468  1467
```

Column 2 has no missing values (code `0x01`, the second row in the column-counts table).
In that SNP position for this sample, 359 indivduals are homozygous allele 1 (`G` according to the `.bim` file), 1004 are heterozygous,
and 577 are homozygous allele 2 (`A`).

This operation also is reasonably fast
```julia
julia> @benchmark columncounts(bf)
BenchmarkTools.Trial: 
  memory estimate:  317.27 KiB
  allocs estimate:  2
  --------------
  minimum time:     37.682 ms (0.00% GC)
  median time:      38.371 ms (0.00% GC)
  mean time:        38.479 ms (0.19% GC)
  maximum time:     47.893 ms (19.91% GC)
  --------------
  samples:          130
  evals/sample:     1
```

In some applications the data are converted to counts of the second allele
|BEDFile|count   |
|------:|--------:|
| 0x00  | 0       |
| 0x01  | missing |
| 0x10  | 1       |
| 0x11  | 2       |

The column means, skipping missing data, are returned by
```julia
julia> mean(bf, dims=1)
1×10150 Array{Float64,2}:
 1.113  1.11237  1.28099  1.11203  1.7376  1.11191  …  1.79966  1.8009  1.79966  1.79955  1.79943

julia> @benchmark mean(bf, dims=1)
BenchmarkTools.Trial: 
  memory estimate:  79.39 KiB
  allocs estimate:  2
  --------------
  minimum time:     57.535 ms (0.00% GC)
  median time:      57.712 ms (0.00% GC)
  mean time:        57.901 ms (0.00% GC)
  maximum time:     59.440 ms (0.00% GC)
  --------------
  samples:          87
  evals/sample:     1
```

A more practical example is to find all the positions in this column with missing values.
Recall that `0x01` indicates a missing value.
```julia
julia> findall(isone.(bf1221))
3-element Array{Int64,1}:
  676
  990
 1044

julia> @benchmark findall(isone.(bf1221)) setup=(bf1221 = BEDColumn(bf, 1221))
BenchmarkTools.Trial: 
  memory estimate:  4.72 KiB
  allocs estimate:  4
  --------------
  minimum time:     12.463 μs (0.00% GC)
  median time:      13.655 μs (0.00% GC)
  mean time:        13.947 μs (0.00% GC)
  maximum time:     1.084 ms (0.00% GC)
  --------------
  samples:          10000
  evals/sample:     1
```
To break this down, the `isone` function applied to a number returns a `Bool`.
```julia
julia> isone(2)
false

julia> isone(1)
true
```
*Dot vectorization* in Julia means that `isone.` applied to an iterator is itself an iterator formed by applying `isone` to each element in turn.
```julia
julia> show(isone.(1:5))
Bool[true, false, false, false, false]
```
and `findall` applied to an iterator of `Bool` values returns the indices of the `true`
values.

The `countcols` method and other column-oriented operations are performed in parallel by assigning a chunk of columns to each thread.

## Instantiating as a count of the second allele

In some operations on GWAS data the data are converted to counts of the second allele.
This is accomplished by indexing `bedvals` with the `BEDColumn`, returning a vector of type
`Union{Missing,UInt8}`, which is the preferred way in v0.7 of representing data
vectors that may contain missing values.
```julia
julia> bedvals[bf1221]
1940-element Array{Union{Missing, UInt8},1}:
 0x01
 0x01
 0x01
 0x01
 0x02
 0x02
 0x01
 0x02
 0x02
    ⋮
 0x02
 0x02
 0x02
 0x02
 0x02
 0x02
 0x02
 0x01
 0x02

julia> sort(unique(bedvals[bf1221]))
4-element Array{Union{Missing, UInt8},1}:
 0x00       
 0x01       
 0x02       
     missing
```
The mean for this column could be calculated as
```julia
julia> using Statistics

julia> mean(bedvals[bf1221])
missing

julia> mean(skipmissing(bedvals[bf1221]))
1.629839958699019

julia> @benchmark mean(skipmissing(bedvals[bf1221])) setup=(bf1221 = BEDColumn(bf, 1221))
BenchmarkTools.Trial: 
  memory estimate:  3.97 KiB
  allocs estimate:  3
  --------------
  minimum time:     50.137 μs (0.00% GC)
  median time:      50.261 μs (0.00% GC)
  mean time:        50.339 μs (0.00% GC)
  maximum time:     136.006 μs (0.00% GC)
  --------------
  samples:          10000
  evals/sample:     1
```

It is slightly faster to use a generator expression
```julia
julia> @benchmark mean(skipmissing(bedvals[v] for v in bf1221))  setup=(bf1221 = BEDColumn(bf, 1221))
BenchmarkTools.Trial: 
  memory estimate:  32 bytes
  allocs estimate:  2
  --------------
  minimum time:     33.203 μs (0.00% GC)
  median time:      33.250 μs (0.00% GC)
  mean time:        33.289 μs (0.00% GC)
  maximum time:     44.130 μs (0.00% GC)
  --------------
  samples:          10000
  evals/sample:     1
```

Notice the difference in memory allocation between this trial and the previous trial.
The generator expression is an iterator whereas the indexing, `bedvals[bf1221]`, actually creates the
array then operates on it.

Of course, this mean could be calculated directly from the column counts
```julia
julia> using LinearAlgebra

julia> const colcounts = columncounts(bf);

julia> cc1221 = colcounts[:, 1221]
4-element Array{Int64,1}:
   75
    3
  567
 1295

julia> dot([0,0,1,2], cc1221) / dot([1,0,1,1], cc1221)
1.629839958699019
```
or with the `mean` method for `BEDFile` defined in this package.
```julia
julia> mean(bf, dims=1)
1×10150 Array{Float64,2}:
 1.113  1.11237  1.28099  1.11203  …  1.8009  1.79966  1.79955  1.79943
```

## Location of the missing values

Some operations require subsetting the rows to only those with complete data.
Discovering whichrows do not have any missing data could be done by iterating across the rows but it is
generally faster to iterate over columns.

Recall that the missing value indicator is 1.
The rows with missing values in column `j` are determined as
```julia
julia> findall(isone.(BEDColumn(bf, 1221)))
3-element Array{Int64,1}:
  676
  990
 1044
```
One way to determine the rows with any missing data is convert the row numbers of missing values to a `BitSet`
and take the union over all the columns. 
```julia
julia> BitSet(findall(isone.(bf1221)))
BitSet([676, 990, 1044])

julia> anymsng = mapreduce(j -> BitSet(findall(isone.(BEDColumn(bf, j)))), union, 1:(size(bf)[2]));

julia> @benchmark mapreduce(j -> BitSet(findall(isone.(BEDColumn($bf, j)))), union, 1:(size($bf)[2]))
BenchmarkTools.Trial: 
  memory estimate:  54.39 MiB
  allocs estimate:  149407
  --------------
  minimum time:     132.531 ms (0.00% GC)
  median time:      134.277 ms (0.00% GC)
  mean time:        134.359 ms (0.00% GC)
  maximum time:     141.487 ms (0.00% GC)
  --------------
  samples:          38
  evals/sample:     1
```
The bad news here is that only a few rows (288, to be exact) don't have any missing data (recall that there are 1940 rows)
```julia
julia> length(anymsng)
1652
```

An alternative is to use `missingpos` to obtain a `SparseMatrixCSC` indicating the positions of the missing values
```julia
julia> msngpos = missingpos(bf)
1940×10150 SparseMatrixCSC{Int8,Int32} with 33922 stored entries:
  [702  ,     1]  =  1
  [949  ,     1]  =  1
  [914  ,     3]  =  1
  [949  ,     3]  =  1
  [1604 ,     3]  =  1
  [1891 ,     3]  =  1
  [81   ,     4]  =  1
  [990  ,     4]  =  1
  [1882 ,     4]  =  1
  ⋮
  [1848 , 10150]  =  1
  [1851 , 10150]  =  1
  [1853 , 10150]  =  1
  [1860 , 10150]  =  1
  [1873 , 10150]  =  1
  [1886 , 10150]  =  1
  [1894 , 10150]  =  1
  [1897 , 10150]  =  1
  [1939 , 10150]  =  1
julia> @time missingpos(bf);
  0.217169 seconds (60.93 k allocations: 48.183 MiB, 1.25% gc time)
```
which can then be reduced using matrix multiplication
```julia
julia> findall(iszero, msngpos * ones(Int, size(msngpos, 2)))'  # rows with no missing data
1×288 LinearAlgebra.Adjoint{Int64,Array{Int64,1}}:
 2  5  11  22  30  37  38  53  56  59  63  65  67  …  1880  1885  1902  1904  1910  1915  1926  1928

julia> @time findall(iszero, msngpos * ones(Int, size(msngpos, 2)))';
  0.000128 seconds (22 allocations: 103.422 KiB)
```


[docs-latest-img]: https://img.shields.io/badge/docs-latest-blue.svg
[docs-latest-url]: https://dmbates.github.io/BEDFiles.jl/latest

[docs-stable-img]: https://img.shields.io/badge/docs-stable-blue.svg
[docs-stable-url]: https://dmbates.github.io/BEDFiles.jl/stable

[travis-img]: https://travis-ci.org/dmbates/BEDFiles.jl.svg?branch=master
[travis-url]: https://travis-ci.org/dmbates/BEDFiles.jl

[appveyor-img]: https://ci.appveyor.com/api/projects/status/lr3tqmbam8sw6714/branch/master?svg=true
[appveyor-url]: https://ci.appveyor.com/project/dmbates/mixedmodels-jl/branch/master

[coveralls-img]: https://coveralls.io/repos/github/dmbates/BEDFiles.jl/badge.svg?branch=master
[coveralls-url]: https://coveralls.io/github/dmbates/BEDFiles.jl?branch=master

[issues-url]: https://github.com/dmbates/BEDFiles.jl/issues

[pkg-0.7-img]: http://pkg.julialang.org/badges/BEDFiles_0.7.svg
[pkg-0.7-url]: http://pkg.julialang.org/?pkg=BEDFiles
