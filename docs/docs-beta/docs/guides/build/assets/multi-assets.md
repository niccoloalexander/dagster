---
title: "Multi-asset"
description: A multi-asset represents a set of assets that are all updated by the same op or graph.
---

A multi-asset represents a set of asset definitions that are all updated by the same [op](/guides/build/ops) or [graph](/guides/build/graphs).

## Relevant APIs

| Name                                        | Description                              |
| ------------------------------------------- | ---------------------------------------- |
| <PyObject section="assets" module="dagster" object="multi_asset" decorator /> | A decorator used to define multi-assets. |

## Overview

When working with [asset definitions](/guides/build/assets/defining-assets), it's sometimes inconvenient or impossible to map each persisted asset to a unique [op](/guides/build/ops) or [graph](/guides/build/graphs). A multi-asset is a way to define a single op or graph that will produce the contents of multiple data assets at once.

Multi-assets may be useful in the following scenarios:

- A single call to an API results in multiple tables being updated (e.g. Airbyte, Fivetran, dbt).
- The same in-memory object is used to compute multiple assets.

## Defining multi-assets

The function responsible for computing the contents of any asset is an [op](/guides/build/ops). Multi-assets are responsible for updating multiple assets, so the underlying op will have multiple outputs -- one for each associated asset.

### A basic multi-asset

The easiest way to create a multi-asset is with the <PyObject section="assets" module="dagster" object="multi_asset" decorator /> decorator. This decorator functions similarly to the <PyObject section="assets" module="dagster" object="asset" decorator /> decorator, but requires a `specs` parameter specifying each asset that the function materializes.

{/* TODO convert to <CodeExample> */}
```python file=/concepts/assets/multi_assets.py startafter=start_basic_multi_asset endbefore=end_basic_multi_asset
from dagster import AssetSpec, multi_asset


@multi_asset(specs=[AssetSpec("users"), AssetSpec("orders")])
def my_function():
    # some code that writes out data to the users table and the orders table
    ...
```

### Conditional materialization

In some cases, an asset may not need to be updated in storage each time the decorated function is executed. You can use the `skippable` parameter along with `yield` syntax and <PyObject section="assets" module="dagster" object="MaterializeResult" /> to implement this behavior.

If the `skippable` parameter is set to `True` on an <PyObject section="assets" module="dagster" object="AssetSpec" />, and your function does not `yield` a <PyObject section="assets" module="dagster" object="MaterializeResult" /> object for that asset, then:

- No asset materialization event will be created
- Downstream assets in the same run will not be materialized
- Asset sensors monitoring the asset will not trigger

{/* TODO convert to <CodeExample> */}
```python file=/concepts/assets/multi_asset_conditional_materialization.py
import random

from dagster import AssetSpec, MaterializeResult, asset, multi_asset


@multi_asset(
    specs=[AssetSpec("asset1", skippable=True), AssetSpec("asset2", skippable=True)]
)
def assets_1_and_2():
    if random.randint(1, 10) < 5:
        yield MaterializeResult(asset_key="asset1")

    if random.randint(1, 10) < 5:
        yield MaterializeResult(asset_key="asset2")


@asset(deps=["asset1"])
def downstream1():
    """Will not run when assets_1_and_2 doesn't materialize asset1."""


@asset(deps=["asset2"])
def downstream2():
    """Will not run when assets_1_and_2 doesn't materialize asset2."""
```

### Subsetting multi-assets

By default, it is assumed that the computation inside of a multi-asset will always produce the contents all of the associated assets. This means that attempting to execute a set of assets that produces some, but not all, of the assets defined by a given multi-asset will result in an error.

Sometimes, the underlying computation is sufficiently flexible to allow for computing an arbitrary subset of the assets associated with it. In these cases, set the `skippable` attribute of the asset specs to `True`, and set the `can_subset` parameter of the decorator to `True`.

Inside the body of the function, we can use `context.selected_asset_keys` to find out which assets should be materialized.

{/* TODO convert to <CodeExample> */}
```python file=/concepts/assets/multi_assets.py startafter=start_subsettable_multi_asset endbefore=end_subsettable_multi_asset
from dagster import AssetExecutionContext, AssetSpec, MaterializeResult, multi_asset


@multi_asset(
    specs=[AssetSpec("asset1", skippable=True), AssetSpec("asset2", skippable=True)],
    can_subset=True,
)
def split_actions(context: AssetExecutionContext):
    if "asset1" in context.op_execution_context.selected_asset_keys:
        yield MaterializeResult(asset_key="asset1")
    if "asset2" in context.op_execution_context.selected_asset_keys:
        yield MaterializeResult(asset_key="asset2")
```

### Dependencies inside multi-assets

Assets defined within multi-assets can have dependencies on upstream assets. These dependencies can be expressed using the `deps` attribute on <PyObject section="assets" module="dagster" object="AssetSpec" />.

{/* TODO convert to <CodeExample> */}
```python file=/concepts/assets/multi_assets.py startafter=start_asset_deps_multi_asset endbefore=end_asset_deps_multi_asset
from dagster import AssetKey, AssetSpec, asset, multi_asset


@asset
def a(): ...


@asset
def b(): ...


@multi_asset(specs=[AssetSpec("c", deps=["b"]), AssetSpec("d", deps=["a"])])
def my_complex_assets(): ...
```

### Multi-asset code versions

Multi-assets may assign different code versions for each of their outputs:

{/* TODO convert to <CodeExample> */}
```python file=/concepts/assets/code_versions.py startafter=start_multi_asset endbefore=end_multi_asset
@multi_asset(
    specs=[AssetSpec(key="a", code_version="1"), AssetSpec(key="b", code_version="2")]
)
def multi_asset_with_versions():
    with open("data/a.json", "w") as f:
        json.dump(100, f)
        yield MaterializeResult("a")
    with open("data/b.json", "w") as f:
        json.dump(200, f)
        yield MaterializeResult("b")
```

Just as with regular assets, these versions are attached to the `AssetMaterialization` objects for each of the constituent assets and represented in the UI.
