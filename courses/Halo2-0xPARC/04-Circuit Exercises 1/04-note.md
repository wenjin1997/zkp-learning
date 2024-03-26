# 范围检查 Range Check
## 方法一：表达式
代码见[example1.rs](/courses/Halo2-0xPARC/halo2-examples/src/range_check/example1.rs)。

范围检查，检查 `value` 是否在[1, RANGE - 1]范围内。思想是如果 `value` 在此范围内，则下列表达式为0:
```
(value) * (1 - value) * (2 - value) * ... * (RANGE - 1 - value)
```
电路中用一个范围检查的 selector `q_range_check` 来考虑是否启用范围检查。

```rust
/// This helper checks that the value witnessed in a given cell is within a given range.
/// 
///     value   |   q_range_check
///     -------------------------
///       v     |       1
/// 

#[derive(Debug, Clone)]
/// A range-constrained value in the circuit produced by the RangeCheckConfig.
struct RangeConstrained<F: FieldExt, const RANGE: usize>(AssignedCell<Assigned<F>, F>);

#[derive(Debug, Clone)]
struct RangeCheckConfig<F: FieldExt, const RANGE: usize> {
    value: Column<Advice>,
    q_range_check: Selector,
    _marker: PhantomData<F>,
}
```
接下来实现 `RangeCheckConfig` ，首先是 `configure` 方法：
```rust
pub fn configure(meta: &mut ConstraintSystem<F>, value: Column<Advice>) -> Self {
    let q_range_check = meta.selector();

    meta.create_gate("range check", |meta| {
        //      value   |   q_range_check
        //      -------------------------
        //         v    |       1

        let q = meta.query_selector(q_range_check);
        let value = meta.query_advice(value, Rotation::cur());
        
        // Given a range R and a value v, returns the expression
        // (v) * (1 - v) * (2 - v) * ... * (R - 1 - v)
        let range_check = |range: usize, value: Expression<F>| {
            assert!(range > 0);
            (1..range).fold(value.clone(), |expr, i| {
                expr * (Expression::Constant(F::from(i as u64)) - value.clone())
            })
        };

        Constraints::with_selector(q, [("range check", range_check(RANGE, value))])
    });

    Self {
        q_range_check,
        value,
        _marker: PhantomData,
    }
}
```
接下来是 `assign` 方法：
```rust
pub fn assign(
    &self,
    mut layouter: impl Layouter<F>,
    value: Value<Assigned<F>>,
) -> Result<RangeConstrained<F, RANGE>, Error> {
    layouter.assign_region(
        || "Assign value", 
        |mut region| {
            let offset = 0;

            // Enable q_range_check
            self.q_range_check.enable(&mut region, offset)?;

            // Assign value
            region
                .assign_advice(|| "value", self.value, offset, || value)
                .map(RangeConstrained)
        },
    )
}
```
画图可以看到此电路布局如下：
![](/courses/Halo2-0xPARC/halo2-examples/range-check-1-layout.png){:height="40%" width="40%"}


## 方法二：Expression + Lookup
代码见
* lookup table 定义：[table.rs](/courses/Halo2-0xPARC/halo2-examples/src/range_check/example2/table.rs)
* 例子：[example2.rs](/courses/Halo2-0xPARC/halo2-examples/src/range_check/example2.rs)

思想是对于小范围的检查用方法一的Expression，而对于大范围，使用查找表方法。

要用到 lookup table，需要用到 `use halo2::proofs::{plonk::TableColumn};`  这个结构， `TableColumn` 结构定义如下：
```rust
/// A fixed column of a lookup table.
///
/// A lookup table can be loaded into this column via [`Layouter::assign_table`]. Columns
/// can currently only contain a single table, but they may be used in multiple lookup
/// arguments via [`ConstraintSystem::lookup`].
///
/// Lookup table columns are always "encumbered" by the lookup arguments they are used in;
/// they cannot simultaneously be used as general fixed columns.
///
/// [`Layouter::assign_table`]: crate::circuit::Layouter::assign_table
#[derive(Clone, Copy, Debug, Eq, PartialEq, Hash)]
pub struct TableColumn {
    /// The fixed column that this table column is stored in.
    ///
    /// # Security
    ///
    /// This inner column MUST NOT be exposed in the public API, or else chip developers
    /// can load lookup tables into their circuits without default-value-filling the
    /// columns, which can cause soundness bugs.
    inner: Column<Fixed>,
}
```

先定义一个查找表，这里的 `value` 就是 `TableColumn` 结构。
```rust
/// A lookup table of values from 0..RANGE.
pub(super) struct RangeTableConfig<F: FieldExt, const RANGE: usize> {
    pub(super) value: TableColumn,
    _marker: PhantomData<F>,
}
```
接着为 `RangeTableConfig` 实现 `configure` 和 `load` 两个方法：
```rust
impl<F: FieldExt, const RANGE: usize> RangeTableConfig<F, RANGE> {
    pub(super) fn configure(meta: &mut ConstraintSystem<F>) -> Self {
        let value = meta.lookup_table_column();

        Self {
            value,
            _marker: PhantomData<F>,
        }
    }

    pub(super) fn load(&self, layouter: &mut impl Layouter<F>) -> Result<(), Error> {
        layouter.assign_table(
            || "load range-check table", 
            |mut table| {
                let mut offset = 0;
                for value in 0..RANGE {
                    table.assign_cell(
                        || "num_bits", 
                        self.value, 
                        offset, 
                        ||Value::known(F::from(value as u64)),
                    )?;
                    offset += 1;
                }

                Ok(())
            },
        )
    }
}
```

在 `RangeCheckConfig` 结构中就有两个选择器，看是用表达式检查还是用lookup。
```rust
#[derive(Debug, Clone)]
struct RangeCheckConfig<F: FieldExt, const RANGE: usize, const LOOKUP_RANGE: usize> {
    q_range_check: Selector,
    q_lookup: Selector,
    value: Column<Advice>,
    table: RangeTableConfig<F, LOOKUP_RANGE>,
}
```
此电路布局如下：
![](/courses/Halo2-0xPARC/halo2-examples/range-check-2-layout.png){:height="40%" width="40%"}
### 方法三：