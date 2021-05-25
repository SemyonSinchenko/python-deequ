# PyDeequ 

PyDeequ is a Python API for [Deequ](https://github.com/awslabs/deequ), a library built on top of Apache Spark for defining "unit tests for data", which measure data quality in large datasets. PyDeequ is written to support usage of Deequ in Python.

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) ![Coverage](https://img.shields.io/badge/coverage-90%25-green)

There are 4 main components of Deequ, and they are: 
- Metrics Computation: 
    - `Profiles` leverages Analyzers to analyze each column of a dataset. 
    - `Analyzers` serve here as a foundational module that computes metrics for data profiling and validation at scale. 
- Constraint Suggestion: 
    - Specify rules for various groups of Analyzers to be run over a dataset to return back a collection of constraints suggested to run in a Verification Suite.
- Constraint Verification: 
    - Perform data validation on a dataset with respect to various constraints set by you.   
- Metrics Repository
    - Allows for persistence and tracking of Deequ runs over time. 

![](imgs/pydeequ_architecture.jpg)

## 🎉 Announcements 🎉
- We've release a blogpost on integrating PyDeequ onto AWS leveraging services such as AWS Glue, Athena, and SageMaker! Check it out: [Monitor data quality in your data lake using PyDeequ and AWS Glue](https://aws.amazon.com/blogs/big-data/monitor-data-quality-in-your-data-lake-using-pydeequ-and-aws-glue/).
- Check out the [PyDeequ Release Announcement Blogpost](https://aws.amazon.com/blogs/big-data/testing-data-quality-at-scale-with-pydeequ/) with a tutorial walkthrough the Amazon Reviews dataset!
- Join the PyDeequ community on [PyDeequ Slack](https://join.slack.com/t/pydeequ/shared_invite/zt-qopmmfgm-ajKRyxx0HgCiK50b9JhAFg) to chat with the devs!

## Quickstart

The following will quickstart you with some basic usage. For more in-depth examples, take a look in the [`tutorials/`](tutorials/) directory for executable Jupyter notebooks of each module. For documentation on supported interfaces, view the [`documentation`](https://pydeequ.readthedocs.io/).

### Installation

You can install [PyDeequ via pip](https://pypi.org/project/pydeequ/).

```
pip install pydeequ
``` 

### Set up a PySpark session 
```python
from pyspark.sql import SparkSession, Row
import pydeequ

spark = (SparkSession
    .builder
    .config("spark.jars.packages", pydeequ.deequ_maven_coord)
    .config("spark.jars.excludes", pydeequ.f2j_maven_coord)
    .getOrCreate())

df = spark.sparkContext.parallelize([
            Row(a="foo", b=1, c=5),
            Row(a="bar", b=2, c=6),
            Row(a="baz", b=3, c=None)]).toDF()
```

### Analyzers 

```python
from pydeequ.analyzers import *

analysisResult = AnalysisRunner(spark) \
                    .onData(df) \
                    .addAnalyzer(Size()) \
                    .addAnalyzer(Completeness("b")) \
                    .run()
                    
analysisResult_df = AnalyzerContext.successMetricsAsDataFrame(spark, analysisResult)
analysisResult_df.show()
```

### Profile 

```python
from pydeequ.profiles import *

result = ColumnProfilerRunner(spark) \
    .onData(df) \
    .run()

for col, profile in result.profiles.items():
    print(profile)
```

### Constraint Suggestions 

```python
from pydeequ.suggestions import *

suggestionResult = ConstraintSuggestionRunner(spark) \
             .onData(df) \
             .addConstraintRule(DEFAULT()) \
             .run()

# Constraint Suggestions in JSON format
print(suggestionResult) 
```

### Constraint Verification 

```python
from pydeequ.checks import *
from pydeequ.verification import *

check = Check(spark, CheckLevel.Warning, "Review Check")

checkResult = VerificationSuite(spark) \
    .onData(df) \
    .addCheck(
        check.hasSize(lambda x: x >= 3) \
        .hasMin("b", lambda x: x == 0) \
        .isComplete("c")  \
        .isUnique("a")  \
        .isContainedIn("a", ["foo", "bar", "baz"]) \
        .isNonNegative("b")) \
    .run()
    
checkResult_df = VerificationResult.checkResultsAsDataFrame(spark, checkResult)
checkResult_df.show()
```

### Repository 

Save to a Metrics Repository by adding the `useRepository()` and `saveOrAppendResult()` calls to your Analysis Runner. 
```python
from pydeequ.repository import *
from pydeequ.analyzers import *

metrics_file = FileSystemMetricsRepository.helper_metrics_file(spark, 'metrics.json')
repository = FileSystemMetricsRepository(spark, metrics_file)
key_tags = {'tag': 'pydeequ hello world'}
resultKey = ResultKey(spark, ResultKey.current_milli_time(), key_tags)

analysisResult = AnalysisRunner(spark) \
    .onData(df) \
    .addAnalyzer(ApproxCountDistinct('b')) \
    .useRepository(repository) \
    .saveOrAppendResult(resultKey) \
    .run()
```

To load previous runs, use the `repository` object to load previous results back in. 

```python
result_metrep_df = repository.load() \
    .before(ResultKey.current_milli_time()) \ 
    .forAnalyzers([ApproxCountDistinct('b')]) \
    .getSuccessMetricsAsDataFrame()
```

## [Contributing](https://github.com/awslabs/python-deequ/blob/master/CONTRIBUTING.md)
Please refer to the [contributing doc](https://github.com/awslabs/python-deequ/blob/master/CONTRIBUTING.md) for how to contribute to PyDeequ. 

## [License](https://github.com/awslabs/python-deequ/blob/master/LICENSE)

This library is licensed under the Apache 2.0 License.

