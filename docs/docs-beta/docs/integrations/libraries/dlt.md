---
layout: Integration
status: published
name: dlt
title: Dagster & dlt
sidebar_label: dlt
excerpt: Easily ingest and replicate data between systems with dlt through Dagster.
date: 2024-08-30
apireflink: https://docs.dagster.io/_apidocs/libraries/dagster-dlt
docslink: https://docs.dagster.io/integrations/dlt
partnerlink: https://www.getdbt.com/
logo: /integrations/dlthub.jpeg
categories:
  - ETL
enabledBy:
enables:
tags: [dagster-supported, etl]
sidebar_custom_props: 
  logo: images/integrations/dlthub.jpeg
---

This integration allows you to use [dlt](https://dlthub.com/) to easily ingest and replicate data between systems through Dagster.

### Installation

```bash
pip install dagster-dlt
```

### Example

<CodeExample filePath="integrations/dlt.py" language="python" />

### About dlt

[Data Load Tool (dlt)](https://dlthub.com/) is an open source library for creating efficient data pipelines. It offers features like secret management, data structure conversion, incremental updates, and pre-built sources and destinations, simplifying the process of loading messy data into well-structured datasets.
