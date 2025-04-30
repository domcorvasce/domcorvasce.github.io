+++
title = 'To Spark or not to Spark'
date = 2025-04-30T18:45:21+01:00
draft = false
readtime = true
toc = false
summary = "A brief reflection about when I would use Spark"
+++

Last night I joined a [Codemotion meetup](https://community.codemotion.com/codemotion-italy/meetups/aperitech-bari---ai-data-intelligence-dallintuizione-dei-ml-alla-gestione-dei-dati) about AI & Data Intelligence, and the second of the two speakers discussed PySpark for processing large amount of data.

The talk had an introductory flavour but I was struck by a question that arose during the Q&A session: **"when is PySpark the right tool for the job?"**.

The answer to this question might have seemed trivial when the only alternative for data processing was Pandas, but nowadays tools such as [Polars](https://pola.rs) and [DuckDB](https://duckdb.org/) should make you question whether Spark is the right tool for the job.

Spark can _handle datasets larger than a single machine's main memory_ simply by engaging multiple machines. It splits both the data and the transformations on the data across multiple nodes (_workers_) and processes (_executors_).

![](https://spark.apache.org/docs/latest/img/cluster-overview.png)

There is a price to pay for this setup: the network bandwidth and latency needed to the transfer data and coordinate work across multiple nodes, additional complexity in debugging issues, poor ergonomics for local development.

Most often than not, the price is not worth paying if the dataset fits into main memory, or you are dealing with one-time data processing jobs (e.g. for research projects).

Even if the dataset is larger than memory, it is worthwhile evaluating whether "scaling out vertically" (i.e. renting a single machine with as much main memory and CPU cores as possible) is more cost effective than "scaling out horizontally" (i.e. distributing data across multiple nodes).
