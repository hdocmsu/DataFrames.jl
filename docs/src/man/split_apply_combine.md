# The Split-Apply-Combine Strategy

Many data analysis tasks involve splitting a data set into groups, applying some
functions to each of the groups and then combining the results. A standardized
framework for handling this sort of computation is described in the paper
"[The Split-Apply-Combine Strategy for Data Analysis](http://www.jstatsoft.org/v40/i01)",
written by Hadley Wickham.

The DataFrames package supports the split-apply-combine strategy through the
`groupby` function followed by `combine`, `select`/`select!` or `transform`/`transform!`.

In order to perform operations by groups you first need to create a `GroupedDataFrame`
object from your data frame using the `groupby` function that takes two arguments:
(1) a data frame to be grouped, and (2) a set of columns to group by.

Operations can then be applied on each group using one of the following functions:
* `combine`: does not put restrictions on number of rows returned, the order of rows
  is specified by the order of groups in `GroupedDataFrame`; it is typically used
  to compute summary statistics by group;
* `select`: return a data frame with the number and order of rows exactly the same
  as the source data frame, including only new calculated columns;
  `select!` is an in-place version of `select`;
* `transform`: return a data frame with the number and order of rows exactly the same
  as the source data frame, including all columns from the source and new calculated columns;
  `transform!` is an in-place version of `transform`.

All these functions take a specification of one or more functions to apply to
each subset of the `DataFrame`. This specification can be of the following forms:
1. standard column selectors (integers, symbols, vectors of integers, vectors of symbols,
   `All`, `:`, `Between`, `Not` and regular expressions)
2. a `cols => function` pair indicating that `function` should be called with
   positional arguments holding columns `cols`, which can be a any valid column selector
3. a `cols => function => target_col` form additionally
   specifying the name of the target column (this assumes that `function` returns a single
   value or a vector)
4. a `col => target_col` pair, which renames the column `col` to `target_col`
5. a `nrow` or `nrow => target_col` form which efficiently computes the number of rows
   in a group (without `target_col` the new column is called `:nrow`)
6. several arguments of the forms given above, or vectors thereof
7. a function which will be called with a `SubDataFrame` corresponding to each group;
   this form should be avoided due to its poor performance unless a very large
   number of columns are processed (in which case `SubDataFrame` avoids excessive
   compilation)

As a special rule that applies to `cols => function` syntax, if `cols` is wrapped
in an `AsTable` object then a `NamedTuple` containing columns selected by `cols` is
passed to `function`.

In all of these cases, `function` can return either a single row or multiple rows.
`function` can always generate a single column by returning a single value or a vector.
Additionally, if `combine` is passed exactly one `function`, `cols => function`,
or `cols => function => outcol` as a first argument
and `target_col` is not specified,
`function` can return multiple columns in the form of an `AbstractDataFrame`,
`AbstractMatrix`, `NamedTuple` or `DataFrameRow`.

`select`/`select!` and `transform`/`transform!` always return a `DataFrame`
with the same number of rows as the source.
For `combine`, the shape of the resulting `DataFrame` is determined
according to the following rules:
- a single value produces a single row and column per group
- a named tuple or `DataFrameRow` produces a single row and one column per field
- a vector produces a single column with one row per entry
- a named tuple of vectors produces one column per field with one row per entry in the vectors
- a `DataFrame` or a matrix produces as many rows and columns as it contains;
  note that this option should be avoided due to its poor performance when the number
  of groups is large

The kind of return value and the number and names of columns must be the same for all groups.

It is allowed to mix single values and vectors if multiple transformations
are requested. In this case single value will be broadcasted to match the length
of columns specified by returned vectors.
As a particular rule, values wrapped in a `Ref` or a `0`-dimensional `AbstractArray`
are unwrapped and then broadcasted.

If a single value or a vector is returned by the `function` and `target_col` is not
provided, it is generated automatically, by concatenating source column name and
`function` name where possible (see examples below).

We show several examples of the `by` function applied to the `iris` dataset below:

```jldoctest sac
julia> using DataFrames, CSV, Statistics

julia> iris = DataFrame(CSV.File(joinpath(dirname(pathof(DataFrames)), "../docs/src/assets/iris.csv")))
150×5 DataFrame
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species        │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ String         │
├─────┼─────────────┼────────────┼─────────────┼────────────┼────────────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ Iris-setosa    │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ Iris-setosa    │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ Iris-setosa    │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ Iris-setosa    │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ Iris-setosa    │
│ 6   │ 5.4         │ 3.9        │ 1.7         │ 0.4        │ Iris-setosa    │
│ 7   │ 4.6         │ 3.4        │ 1.4         │ 0.3        │ Iris-setosa    │
⋮
│ 143 │ 5.8         │ 2.7        │ 5.1         │ 1.9        │ Iris-virginica │
│ 144 │ 6.8         │ 3.2        │ 5.9         │ 2.3        │ Iris-virginica │
│ 145 │ 6.7         │ 3.3        │ 5.7         │ 2.5        │ Iris-virginica │
│ 146 │ 6.7         │ 3.0        │ 5.2         │ 2.3        │ Iris-virginica │
│ 147 │ 6.3         │ 2.5        │ 5.0         │ 1.9        │ Iris-virginica │
│ 148 │ 6.5         │ 3.0        │ 5.2         │ 2.0        │ Iris-virginica │
│ 149 │ 6.2         │ 3.4        │ 5.4         │ 2.3        │ Iris-virginica │
│ 150 │ 5.9         │ 3.0        │ 5.1         │ 1.8        │ Iris-virginica │

julia> gdf = groupby(iris, :Species)
GroupedDataFrame with 3 groups based on key: Species
First Group (50 rows): Species = "Iris-setosa"
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species     │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ String      │
├─────┼─────────────┼────────────┼─────────────┼────────────┼─────────────┤
│ 1   │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ Iris-setosa │
│ 2   │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ Iris-setosa │
│ 3   │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ Iris-setosa │
│ 4   │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ Iris-setosa │
│ 5   │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ Iris-setosa │
│ 6   │ 5.4         │ 3.9        │ 1.7         │ 0.4        │ Iris-setosa │
│ 7   │ 4.6         │ 3.4        │ 1.4         │ 0.3        │ Iris-setosa │
⋮
│ 43  │ 4.4         │ 3.2        │ 1.3         │ 0.2        │ Iris-setosa │
│ 44  │ 5.0         │ 3.5        │ 1.6         │ 0.6        │ Iris-setosa │
│ 45  │ 5.1         │ 3.8        │ 1.9         │ 0.4        │ Iris-setosa │
│ 46  │ 4.8         │ 3.0        │ 1.4         │ 0.3        │ Iris-setosa │
│ 47  │ 5.1         │ 3.8        │ 1.6         │ 0.2        │ Iris-setosa │
│ 48  │ 4.6         │ 3.2        │ 1.4         │ 0.2        │ Iris-setosa │
│ 49  │ 5.3         │ 3.7        │ 1.5         │ 0.2        │ Iris-setosa │
│ 50  │ 5.0         │ 3.3        │ 1.4         │ 0.2        │ Iris-setosa │
⋮
Last Group (50 rows): Species = "Iris-virginica"
│ Row │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species        │
│     │ Float64     │ Float64    │ Float64     │ Float64    │ String         │
├─────┼─────────────┼────────────┼─────────────┼────────────┼────────────────┤
│ 1   │ 6.3         │ 3.3        │ 6.0         │ 2.5        │ Iris-virginica │
│ 2   │ 5.8         │ 2.7        │ 5.1         │ 1.9        │ Iris-virginica │
│ 3   │ 7.1         │ 3.0        │ 5.9         │ 2.1        │ Iris-virginica │
│ 4   │ 6.3         │ 2.9        │ 5.6         │ 1.8        │ Iris-virginica │
│ 5   │ 6.5         │ 3.0        │ 5.8         │ 2.2        │ Iris-virginica │
│ 6   │ 7.6         │ 3.0        │ 6.6         │ 2.1        │ Iris-virginica │
│ 7   │ 4.9         │ 2.5        │ 4.5         │ 1.7        │ Iris-virginica │
⋮
│ 43  │ 5.8         │ 2.7        │ 5.1         │ 1.9        │ Iris-virginica │
│ 44  │ 6.8         │ 3.2        │ 5.9         │ 2.3        │ Iris-virginica │
│ 45  │ 6.7         │ 3.3        │ 5.7         │ 2.5        │ Iris-virginica │
│ 46  │ 6.7         │ 3.0        │ 5.2         │ 2.3        │ Iris-virginica │
│ 47  │ 6.3         │ 2.5        │ 5.0         │ 1.9        │ Iris-virginica │
│ 48  │ 6.5         │ 3.0        │ 5.2         │ 2.0        │ Iris-virginica │
│ 49  │ 6.2         │ 3.4        │ 5.4         │ 2.3        │ Iris-virginica │
│ 50  │ 5.9         │ 3.0        │ 5.1         │ 1.8        │ Iris-virginica │

julia> combine(gdf, :PetalLength => mean)
3×2 DataFrame
│ Row │ Species         │ PetalLength_mean │
│     │ String          │ Float64          │
├─────┼─────────────────┼──────────────────┤
│ 1   │ Iris-setosa     │ 1.464            │
│ 2   │ Iris-versicolor │ 4.26             │
│ 3   │ Iris-virginica  │ 5.552            │

julia> combine(gdf, nrow)
3×2 DataFrame
│ Row │ Species         │ nrow  │
│     │ String          │ Int64 │
├─────┼─────────────────┼───────┤
│ 1   │ Iris-setosa     │ 50    │
│ 2   │ Iris-versicolor │ 50    │
│ 3   │ Iris-virginica  │ 50    │

julia> combine(gdf, nrow, :PetalLength => mean => :mean)
3×3 DataFrame
│ Row │ Species         │ nrow  │ mean    │
│     │ String          │ Int64 │ Float64 │
├─────┼─────────────────┼───────┼─────────┤
│ 1   │ Iris-setosa     │ 50    │ 1.464   │
│ 2   │ Iris-versicolor │ 50    │ 4.26    │
│ 3   │ Iris-virginica  │ 50    │ 5.552   │

julia> combine([:PetalLength, :SepalLength] => (p, s) -> (a=mean(p)/mean(s), b=sum(p)),
               gdf) # multiple columns are passed as arguments
3×3 DataFrame
│ Row │ Species         │ a        │ b       │
│     │ String          │ Float64  │ Float64 │
├─────┼─────────────────┼──────────┼─────────┤
│ 1   │ Iris-setosa     │ 0.292449 │ 73.2    │
│ 2   │ Iris-versicolor │ 0.717655 │ 213.0   │
│ 3   │ Iris-virginica  │ 0.842744 │ 277.6   │

julia> combine(gdf,
               AsTable([:PetalLength, :SepalLength]) =>
               x -> std(x.PetalLength) / std(x.SepalLength)) # passing a NamedTuple
3×2 DataFrame
│ Row │ Species         │ PetalLength_SepalLength_function │
│     │ String          │ Float64                          │
├─────┼─────────────────┼──────────────────────────────────┤
│ 1   │ Iris-setosa     │ 0.492245                         │
│ 2   │ Iris-versicolor │ 0.910378                         │
│ 3   │ Iris-virginica  │ 0.867923                         │

julia> combine(x -> std(x.PetalLength) / std(x.SepalLength), gdf) # passing a SubDataFrame
3×2 DataFrame
│ Row │ Species         │ PetalLength_SepalLength_function │
│     │ String          │ Float64                          │
├─────┼─────────────────┼──────────────────────────────────┤
│ 1   │ Iris-setosa     │ 0.492245                         │
│ 2   │ Iris-versicolor │ 0.910378                         │
│ 3   │ Iris-virginica  │ 0.867923                         │

julia> combine(gdf, 1:2 => cor, nrow)
3×3 DataFrame
│ Row │ Species         │ SepalLength_SepalWidth_cor │ nrow  │
│     │ String          │ Float64                    │ Int64 │
├─────┼─────────────────┼────────────────────────────┼───────┤
│ 1   │ Iris-setosa     │ 0.74678                    │ 50    │
│ 2   │ Iris-versicolor │ 0.525911                   │ 50    │
│ 3   │ Iris-virginica  │ 0.457228                   │ 50    │

```

Contrary to `combine`, the `select` and `transform` functions always return
a data frame with the same number and order of rows as the source.
In the example below
the return values in columns `:SepalLength_SepalWidth_cor` and `:nrow` are
broadcasted to match the number of elements in each group:
```
julia> select(gdf, 1:2 => cor)
150×2 DataFrame
│ Row │ Species        │ SepalLength_SepalWidth_cor │
│     │ String         │ Float64                    │
├─────┼────────────────┼────────────────────────────┤
│ 1   │ Iris-setosa    │ 0.74678                    │
│ 2   │ Iris-setosa    │ 0.74678                    │
│ 3   │ Iris-setosa    │ 0.74678                    │
│ 4   │ Iris-setosa    │ 0.74678                    │
│ 5   │ Iris-setosa    │ 0.74678                    │
│ 6   │ Iris-setosa    │ 0.74678                    │
│ 7   │ Iris-setosa    │ 0.74678                    │
⋮
│ 143 │ Iris-virginica │ 0.457228                   │
│ 144 │ Iris-virginica │ 0.457228                   │
│ 145 │ Iris-virginica │ 0.457228                   │
│ 146 │ Iris-virginica │ 0.457228                   │
│ 147 │ Iris-virginica │ 0.457228                   │
│ 148 │ Iris-virginica │ 0.457228                   │
│ 149 │ Iris-virginica │ 0.457228                   │
│ 150 │ Iris-virginica │ 0.457228                   │

julia> transform(gdf, :Species => x -> chop.(x, head=5, tail=0))
150×6 DataFrame
│ Row │ Species        │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │ Species_function │
│     │ String         │ Float64     │ Float64    │ Float64     │ Float64    │ SubString…       │
├─────┼────────────────┼─────────────┼────────────┼─────────────┼────────────┼──────────────────┤
│ 1   │ Iris-setosa    │ 5.1         │ 3.5        │ 1.4         │ 0.2        │ setosa           │
│ 2   │ Iris-setosa    │ 4.9         │ 3.0        │ 1.4         │ 0.2        │ setosa           │
│ 3   │ Iris-setosa    │ 4.7         │ 3.2        │ 1.3         │ 0.2        │ setosa           │
│ 4   │ Iris-setosa    │ 4.6         │ 3.1        │ 1.5         │ 0.2        │ setosa           │
│ 5   │ Iris-setosa    │ 5.0         │ 3.6        │ 1.4         │ 0.2        │ setosa           │
│ 6   │ Iris-setosa    │ 5.4         │ 3.9        │ 1.7         │ 0.4        │ setosa           │
│ 7   │ Iris-setosa    │ 4.6         │ 3.4        │ 1.4         │ 0.3        │ setosa           │
⋮
│ 143 │ Iris-virginica │ 5.8         │ 2.7        │ 5.1         │ 1.9        │ virginica        │
│ 144 │ Iris-virginica │ 6.8         │ 3.2        │ 5.9         │ 2.3        │ virginica        │
│ 145 │ Iris-virginica │ 6.7         │ 3.3        │ 5.7         │ 2.5        │ virginica        │
│ 146 │ Iris-virginica │ 6.7         │ 3.0        │ 5.2         │ 2.3        │ virginica        │
│ 147 │ Iris-virginica │ 6.3         │ 2.5        │ 5.0         │ 1.9        │ virginica        │
│ 148 │ Iris-virginica │ 6.5         │ 3.0        │ 5.2         │ 2.0        │ virginica        │
│ 149 │ Iris-virginica │ 6.2         │ 3.4        │ 5.4         │ 2.3        │ virginica        │
│ 150 │ Iris-virginica │ 5.9         │ 3.0        │ 5.1         │ 1.8        │ virginica        │
```

The `combine` function also supports the `do` block form. However, as noted above,
this form is slow and should therefore be avoided when performance matters.

```jldoctest sac
julia> combine(gdf) do df
           (m = mean(df.PetalLength), s² = var(df.PetalLength))
       end
3×3 DataFrame
│ Row │ Species         │ m       │ s²        │
│     │ String          │ Float64 │ Float64   │
├─────┼─────────────────┼─────────┼───────────┤
│ 1   │ Iris-setosa     │ 1.464   │ 0.0301061 │
│ 2   │ Iris-versicolor │ 4.26    │ 0.220816  │
│ 3   │ Iris-virginica  │ 5.552   │ 0.304588  │
```

If you only want to split the data set into subsets, use the [`groupby`](@ref) function:

```jldoctest sac
julia> for subdf in groupby(iris, :Species)
           println(size(subdf, 1))
       end
50
50
50
```

To also get the values of the grouping columns along with each group, use the
`pairs` function:

```jldoctest sac
julia> for (key, subdf) in pairs(groupby(iris, :Species))
           println("Number of data points for $(key.Species): $(nrow(subdf))")
       end
Number of data points for Iris-setosa: 50
Number of data points for Iris-versicolor: 50
Number of data points for Iris-virginica: 50
```

The value of `key` in the previous example is a [`DataFrames.GroupKey`](@ref) object,
which can be used in a similar fashion to a `NamedTuple`.

Grouping a data frame using the `groupby` function can be seen as adding a lookup key
to it. Such lookups can be performed efficiently by indexing the resulting
`GroupedDataFrame` with a `Tuple` or `NamedTuple`:
```
julia> df = DataFrame(g = repeat(1:1000, inner=5), x = 1:5000);

julia> gdf = groupby(df, :g)
GroupedDataFrame with 1000 groups based on key: g
First Group (5 rows): g = 1
│ Row │ g     │ x     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 1     │ 1     │
│ 2   │ 1     │ 2     │
│ 3   │ 1     │ 3     │
│ 4   │ 1     │ 4     │
│ 5   │ 1     │ 5     │
⋮
Last Group (5 rows): g = 1000
│ Row │ g     │ x     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 1000  │ 4996  │
│ 2   │ 1000  │ 4997  │
│ 3   │ 1000  │ 4998  │
│ 4   │ 1000  │ 4999  │
│ 5   │ 1000  │ 5000  │

julia> gdf[(g=500,)]
5×2 SubDataFrame
│ Row │ g     │ x     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 500   │ 2496  │
│ 2   │ 500   │ 2497  │
│ 3   │ 500   │ 2498  │
│ 4   │ 500   │ 2499  │
│ 5   │ 500   │ 2500  │

julia> gdf[[(500,), (501,)]]
GroupedDataFrame with 2 groups based on key: g
First Group (5 rows): g = 500
│ Row │ g     │ x     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 500   │ 2496  │
│ 2   │ 500   │ 2497  │
│ 3   │ 500   │ 2498  │
│ 4   │ 500   │ 2499  │
│ 5   │ 500   │ 2500  │
⋮
Last Group (5 rows): g = 501
│ Row │ g     │ x     │
│     │ Int64 │ Int64 │
├─────┼───────┼───────┤
│ 1   │ 501   │ 2501  │
│ 2   │ 501   │ 2502  │
│ 3   │ 501   │ 2503  │
│ 4   │ 501   │ 2504  │
│ 5   │ 501   │ 2505  │
```

In order to apply a function to each non-grouping column of a `GroupedDataFrame` you can write:
```jldoctest sac
julia> gd = groupby(iris, :Species);

julia> combine(gd, valuecols(gd) .=> mean)
3×5 DataFrame
│ Row │ Species         │ SepalLength_mean │ SepalWidth_mean │ PetalLength_mean │ PetalWidth_mean │
│     │ String          │ Float64          │ Float64         │ Float64          │ Float64         │
├─────┼─────────────────┼──────────────────┼─────────────────┼──────────────────┼─────────────────┤
│ 1   │ Iris-setosa     │ 5.006            │ 3.418           │ 1.464            │ 0.244           │
│ 2   │ Iris-versicolor │ 5.936            │ 2.77            │ 4.26             │ 1.326           │
│ 3   │ Iris-virginica  │ 6.588            │ 2.974           │ 5.552            │ 2.026           │

julia> combine(gd, valuecols(gd) .=> (x -> (x .- mean(x)) ./ std(x)) .=> valuecols(gd))
150×5 DataFrame
│ Row │ Species        │ SepalLength │ SepalWidth │ PetalLength │ PetalWidth │
│     │ String         │ Float64     │ Float64    │ Float64     │ Float64    │
├─────┼────────────────┼─────────────┼────────────┼─────────────┼────────────┤
│ 1   │ Iris-setosa    │ 0.266674    │ 0.215209   │ -0.368852   │ -0.410411  │
│ 2   │ Iris-setosa    │ -0.300718   │ -1.09704   │ -0.368852   │ -0.410411  │
│ 3   │ Iris-setosa    │ -0.868111   │ -0.572142  │ -0.945184   │ -0.410411  │
│ 4   │ Iris-setosa    │ -1.15181    │ -0.834592  │ 0.207479    │ -0.410411  │
│ 5   │ Iris-setosa    │ -0.0170218  │ 0.47766    │ -0.368852   │ -0.410411  │
│ 6   │ Iris-setosa    │ 1.11776     │ 1.26501    │ 1.36014     │ 1.45509    │
│ 7   │ Iris-setosa    │ -1.15181    │ -0.0472411 │ -0.368852   │ 0.522342   │
⋮
│ 143 │ Iris-virginica │ -1.23923    │ -0.849621  │ -0.818997   │ -0.458766  │
│ 144 │ Iris-virginica │ 0.333396    │ 0.700782   │ 0.630555    │ 0.997633   │
│ 145 │ Iris-virginica │ 0.176134    │ 1.01086    │ 0.268167    │ 1.72583    │
│ 146 │ Iris-virginica │ 0.176134    │ 0.080621   │ -0.637803   │ 0.997633   │
│ 147 │ Iris-virginica │ -0.452916   │ -1.46978   │ -1.00019    │ -0.458766  │
│ 148 │ Iris-virginica │ -0.138391   │ 0.080621   │ -0.637803   │ -0.0946659 │
│ 149 │ Iris-virginica │ -0.610178   │ 1.32094    │ -0.275415   │ 0.997633   │
│ 150 │ Iris-virginica │ -1.08197    │ 0.080621   │ -0.818997   │ -0.822865  │
```
