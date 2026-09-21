# Large-Scale Data Pipeline & Reporting Automation

An end-to-end data pipeline built with Python and PostgreSQL to process, clean, validate, transform, and publish large operational datasets for analytics and reporting.

## Project Overview

This project was designed to turn raw operational report files into a reliable, analysis-ready dataset.

The workflow handles millions of records and automates key stages of the data lifecycle, including:

- Data ingestion
- Exact duplicate removal
- Data type standardization
- Missing-value handling
- Business-rule transformations
- Data validation
- PostgreSQL loading
- Automated daily processing
- Reporting-ready data preparation

The project focuses on building a repeatable and reliable data workflow rather than performing one-time manual data cleaning.

## Technologies

- Python
- Pandas
- PostgreSQL
- SQL
- PowerShell
- Git
- Windows Task Scheduler
- 
## Business Problem

The source data was generated as large operational report files that required repeated manual processing before it could be used reliably for analysis and reporting.

Key challenges included:

- Large volumes of historical and daily data
- Duplicate records across multiple report files
- Inconsistent data types and formatting
- Missing or incomplete values in important fields
- Repeated business-rule transformations
- The need to integrate new daily data without rebuilding the entire dataset
- The need to keep the analytical database synchronized with newly processed data
- Manual report acquisition and processing created unnecessary operational effort

The goal was to build a reliable pipeline that could process both historical and newly arriving data consistently, validate the results, and publish analysis-ready records to PostgreSQL with minimal manual intervention.
