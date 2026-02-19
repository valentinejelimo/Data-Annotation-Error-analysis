# SQL Analysis Queries for Annotation Platform Data

## Table of Contents
1. [Setup & Data Loading](#setup--data-loading)
2. [Basic Exploratory Queries](#basic-exploratory-queries)
3. [Error Rate Analysis](#error-rate-analysis)
4. [Platform Comparison Queries](#platform-comparison-queries)
5. [Annotator Performance Analysis](#annotator-performance-analysis)
6. [Task Type Analysis](#task-type-analysis)
7. [Temporal Analysis](#temporal-analysis)
8. [Guideline Analysis](#guideline-analysis)
9. [Quality & Confidence Analysis](#quality--confidence-analysis)
10. [Advanced Analytics Queries](#advanced-analytics-queries)

---

## Setup & Data Loading

### Create Table Structure
**Purpose**: Define the table schema for loading the cleaned dataset

```sql
-- Create the main annotations table
CREATE TABLE annotations (
    annotation_id VARCHAR(20) PRIMARY KEY,
    platform VARCHAR(50) NOT NULL,
    project_name VARCHAR(100) NOT NULL,
    annotator_id VARCHAR(50) NOT NULL,
    task_id VARCHAR(50),
    task_type VARCHAR(50) NOT NULL,
    submission_timestamp TIMESTAMP NOT NULL,
    annotation_quality_score DECIMAL(3,2),
    error_flag INTEGER NOT NULL,
    error_type VARCHAR(50) NOT NULL,
    time_spent_seconds DECIMAL(10,2),
    annotator_experience_days INTEGER NOT NULL,
    training_completed VARCHAR(3) NOT NULL,
    guideline_version VARCHAR(20),
    reviewer_id VARCHAR(50),
    confidence_score DECIMAL(3,2),
    month VARCHAR(20),
    time_of_day VARCHAR(20),
    time_is_outlier BOOLEAN,
    guideline_version_imputed BOOLEAN,
    task_id_missing BOOLEAN,
    is_reviewed BOOLEAN
);

-- Create indexes for faster querying
CREATE INDEX idx_platform ON annotations(platform);
CREATE INDEX idx_project ON annotations(project_name);
CREATE INDEX idx_annotator ON annotations(annotator_id);
CREATE INDEX idx_error_flag ON annotations(error_flag);
CREATE INDEX idx_task_type ON annotations(task_type);
CREATE INDEX idx_submission_date ON annotations(submission_timestamp);
CREATE INDEX idx_reviewer ON annotations(reviewer_id);
```

### Load Data (PostgreSQL example)
```sql
-- Load CSV data into table
COPY annotations
FROM '/path/to/annotation_data_cleaned_final.csv'
DELIMITER ','
CSV HEADER;
```

---

## Basic Exploratory Queries

### Query 1: Dataset Overview
**Purpose**: Get high-level statistics about the entire dataset

```sql
SELECT 
    COUNT(*) AS total_annotations,
    COUNT(DISTINCT platform) AS total_platforms,
    COUNT(DISTINCT project_name) AS total_projects,
    COUNT(DISTINCT annotator_id) AS total_annotators,
    COUNT(DISTINCT reviewer_id) AS total_reviewers,
    COUNT(DISTINCT task_type) AS total_task_types,
    MIN(submission_timestamp) AS earliest_submission,
    MAX(submission_timestamp) AS latest_submission,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_spent_seconds,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotations;
```

### Query 2: Platform Distribution
**Purpose**: Understand the volume split between platforms

```sql
SELECT 
    platform,
    COUNT(*) AS total_annotations,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage_of_total,
    COUNT(DISTINCT annotator_id) AS unique_annotators,
    COUNT(DISTINCT project_name) AS projects
FROM annotations
GROUP BY platform
ORDER BY total_annotations DESC;
```

### Query 3: Project Distribution
**Purpose**: See annotation volume by project and platform

```sql
SELECT 
    platform,
    project_name,
    COUNT(*) AS total_annotations,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(PARTITION BY platform), 2) AS pct_within_platform,
    COUNT(DISTINCT annotator_id) AS unique_annotators,
    COUNT(DISTINCT task_type) AS task_types
FROM annotations
GROUP BY platform, project_name
ORDER BY platform, total_annotations DESC;
```

---

## Error Rate Analysis

### Query 4: Overall Error Rate
**Purpose**: Calculate the overall error rate across the dataset

```sql
SELECT 
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    COUNT(*) - SUM(error_flag) AS total_correct,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage
FROM annotations;
```

### Query 5: Error Rate by Platform
**Purpose**: Compare error rates between Platform A and Platform B - KEY INSIGHT QUERY

```sql
SELECT 
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    COUNT(*) - SUM(error_flag) AS total_correct,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotations
GROUP BY platform
ORDER BY error_rate_percentage DESC;
```

### Query 6: Error Type Distribution
**Purpose**: Understand what types of errors are most common

```sql
SELECT 
    error_type,
    COUNT(*) AS occurrences,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage_of_total
FROM annotations
WHERE error_flag = 1  -- Only look at actual errors
GROUP BY error_type
ORDER BY occurrences DESC;
```

### Query 7: Error Type by Platform - CRITICAL INSIGHT
**Purpose**: Identify platform-specific error patterns (shows guideline confusion in Platform A)

```sql
SELECT 
    platform,
    error_type,
    COUNT(*) AS error_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(PARTITION BY platform), 2) AS pct_of_platform_errors
FROM annotations
WHERE error_flag = 1
GROUP BY platform, error_type
ORDER BY platform, error_count DESC;
```

### Query 8: Error Rate by Project
**Purpose**: Identify which projects have the highest error rates

```sql
SELECT 
    platform,
    project_name,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotations
GROUP BY platform, project_name
ORDER BY error_rate_percentage DESC;
```

---

## Platform Comparison Queries

### Query 9: Platform Performance Comparison
**Purpose**: Comprehensive side-by-side platform comparison

```sql
SELECT 
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(confidence_score), 2) AS avg_confidence_score,
    COUNT(DISTINCT annotator_id) AS unique_annotators,
    ROUND(AVG(annotator_experience_days), 0) AS avg_experience_days
FROM annotations
GROUP BY platform
ORDER BY platform;
```

### Query 10: Guideline Confusion Analysis - KEY FINDING
**Purpose**: Quantify the impact of guideline confusion (Platform A's main problem)

```sql
SELECT 
    platform,
    SUM(CASE WHEN error_type = 'guideline_confusion' THEN 1 ELSE 0 END) AS guideline_confusion_errors,
    SUM(error_flag) AS total_errors,
    ROUND(
        SUM(CASE WHEN error_type = 'guideline_confusion' THEN 1 ELSE 0 END) * 100.0 / 
        NULLIF(SUM(error_flag), 0), 
        2
    ) AS pct_errors_from_guideline_confusion
FROM annotations
GROUP BY platform
ORDER BY guideline_confusion_errors DESC;
```

### Query 11: Platform A - Project Switching Impact
**Purpose**: Show how annotators working on multiple projects have higher errors

```sql
-- First, identify annotators who worked on multiple projects
WITH annotator_project_counts AS (
    SELECT 
        annotator_id,
        COUNT(DISTINCT project_name) AS projects_worked
    FROM annotations
    WHERE platform = 'Platform_A'
    GROUP BY annotator_id
)

SELECT 
    CASE 
        WHEN apc.projects_worked = 1 THEN 'Single Project'
        ELSE 'Multiple Projects'
    END AS annotator_type,
    COUNT(*) AS total_annotations,
    SUM(a.error_flag) AS total_errors,
    ROUND(AVG(a.error_flag) * 100, 2) AS error_rate_pct,
    SUM(CASE WHEN a.error_type = 'guideline_confusion' THEN 1 ELSE 0 END) AS guideline_confusion_errors,
    COUNT(DISTINCT a.annotator_id) AS num_annotators
FROM annotations a
JOIN annotator_project_counts apc ON a.annotator_id = apc.annotator_id
WHERE a.platform = 'Platform_A'
GROUP BY 
    CASE 
        WHEN apc.projects_worked = 1 THEN 'Single Project'
        ELSE 'Multiple Projects'
    END
ORDER BY error_rate_pct DESC;
```

---

## Annotator Performance Analysis

### Query 12: Top Performing Annotators
**Purpose**: Identify the best annotators (lowest error rate, minimum 50 annotations)

```sql
SELECT 
    annotator_id,
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds,
    ROUND(AVG(annotator_experience_days), 0) AS avg_experience_days
FROM annotations
GROUP BY annotator_id, platform
HAVING COUNT(*) >= 50  -- Minimum 50 annotations for statistical significance
ORDER BY error_rate_pct ASC, total_annotations DESC
LIMIT 20;
```

### Query 13: Worst Performing Annotators
**Purpose**: Identify annotators who need additional training

```sql
SELECT 
    annotator_id,
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(annotator_experience_days), 0) AS avg_experience_days,
    training_completed,
    STRING_AGG(DISTINCT error_type, ', ') AS common_error_types
FROM annotations
WHERE error_flag = 1
GROUP BY annotator_id, platform, training_completed
HAVING COUNT(*) >= 50
ORDER BY error_rate_pct DESC
LIMIT 20;
```

### Query 14: Experience vs Error Rate
**Purpose**: Show correlation between experience and error rate

```sql
SELECT 
    CASE 
        WHEN annotator_experience_days < 30 THEN '0-29 days (New)'
        WHEN annotator_experience_days < 90 THEN '30-89 days (Intermediate)'
        WHEN annotator_experience_days < 180 THEN '90-179 days (Experienced)'
        ELSE '180+ days (Expert)'
    END AS experience_level,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    COUNT(DISTINCT annotator_id) AS num_annotators
FROM annotations
GROUP BY 
    CASE 
        WHEN annotator_experience_days < 30 THEN '0-29 days (New)'
        WHEN annotator_experience_days < 90 THEN '30-89 days (Intermediate)'
        WHEN annotator_experience_days < 180 THEN '90-179 days (Experienced)'
        ELSE '180+ days (Expert)'
    END
ORDER BY 
    CASE 
        WHEN annotator_experience_days < 30 THEN 1
        WHEN annotator_experience_days < 90 THEN 2
        WHEN annotator_experience_days < 180 THEN 3
        ELSE 4
    END;
```

### Query 15: Training Impact
**Purpose**: Assess whether training completion affects error rates

```sql
SELECT 
    platform,
    training_completed,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    COUNT(DISTINCT annotator_id) AS num_annotators
FROM annotations
GROUP BY platform, training_completed
ORDER BY platform, training_completed;
```

---

## Task Type Analysis

### Query 16: Error Rate by Task Type
**Purpose**: Identify which task types are most challenging

```sql
SELECT 
    task_type,
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds
FROM annotations
GROUP BY task_type, platform
ORDER BY error_rate_pct DESC;
```

### Query 17: Task Complexity Analysis
**Purpose**: Compare task complexity using time spent and error rates

```sql
SELECT 
    task_type,
    COUNT(*) AS total_annotations,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    CASE 
        WHEN AVG(time_spent_seconds) > 100 AND AVG(error_flag) > 0.15 THEN 'High Complexity'
        WHEN AVG(time_spent_seconds) > 100 OR AVG(error_flag) > 0.15 THEN 'Medium Complexity'
        ELSE 'Low Complexity'
    END AS complexity_level
FROM annotations
GROUP BY task_type
ORDER BY avg_time_seconds DESC;
```

### Query 18: Most Common Errors by Task Type
**Purpose**: Understand what errors occur most frequently in each task type

```sql
SELECT 
    task_type,
    error_type,
    COUNT(*) AS error_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(PARTITION BY task_type), 2) AS pct_of_task_errors
FROM annotations
WHERE error_flag = 1
GROUP BY task_type, error_type
ORDER BY task_type, error_count DESC;
```

---

## Temporal Analysis

### Query 19: Error Rate by Month
**Purpose**: Identify if error rates improve or worsen over time

```sql
SELECT 
    month,
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotations
GROUP BY month, platform
ORDER BY 
    CASE month
        WHEN 'January' THEN 1
        WHEN 'February' THEN 2
        WHEN 'March' THEN 3
        WHEN 'April' THEN 4
        WHEN 'May' THEN 5
        WHEN 'June' THEN 6
        WHEN 'July' THEN 7
        WHEN 'August' THEN 8
        WHEN 'September' THEN 9
        WHEN 'October' THEN 10
        WHEN 'November' THEN 11
        WHEN 'December' THEN 12
    END,
    platform;
```

### Query 20: Time of Day Performance
**Purpose**: Check if certain times of day have higher error rates

```sql
SELECT 
    time_of_day,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds,
    COUNT(DISTINCT annotator_id) AS num_annotators
FROM annotations
GROUP BY time_of_day
ORDER BY 
    CASE time_of_day
        WHEN 'Morning' THEN 1
        WHEN 'Afternoon' THEN 2
        WHEN 'Evening' THEN 3
        WHEN 'Night' THEN 4
    END;
```

### Query 21: Weekly Trends
**Purpose**: Analyze if day of week affects performance

```sql
SELECT 
    EXTRACT(DOW FROM submission_timestamp) AS day_of_week,
    CASE EXTRACT(DOW FROM submission_timestamp)
        WHEN 0 THEN 'Sunday'
        WHEN 1 THEN 'Monday'
        WHEN 2 THEN 'Tuesday'
        WHEN 3 THEN 'Wednesday'
        WHEN 4 THEN 'Thursday'
        WHEN 5 THEN 'Friday'
        WHEN 6 THEN 'Saturday'
    END AS day_name,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotations
GROUP BY EXTRACT(DOW FROM submission_timestamp)
ORDER BY day_of_week;
```

---

## Guideline Analysis

### Query 22: Error Rate by Guideline Version
**Purpose**: Identify if certain guideline versions have higher error rates

```sql
SELECT 
    guideline_version,
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    guideline_version_imputed
FROM annotations
GROUP BY guideline_version, platform, guideline_version_imputed
HAVING COUNT(*) >= 100  -- Minimum sample size
ORDER BY platform, error_rate_pct DESC;
```

### Query 23: Guideline Version Evolution
**Purpose**: Track how error rates changed as guidelines were updated

```sql
WITH version_categories AS (
    SELECT 
        guideline_version,
        SUBSTRING(guideline_version FROM 'v([0-9]+)') AS major_version,
        COUNT(*) AS annotations,
        ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct
    FROM annotations
    WHERE guideline_version LIKE 'v%'
    GROUP BY guideline_version
    HAVING COUNT(*) >= 50
)

SELECT 
    major_version,
    COUNT(*) AS num_versions,
    SUM(annotations) AS total_annotations,
    ROUND(AVG(error_rate_pct), 2) AS avg_error_rate_pct,
    MIN(error_rate_pct) AS min_error_rate,
    MAX(error_rate_pct) AS max_error_rate
FROM version_categories
GROUP BY major_version
ORDER BY major_version;
```

---

## Quality & Confidence Analysis

### Query 24: Quality Score Distribution
**Purpose**: Understand the distribution of quality scores

```sql
SELECT 
    CASE 
        WHEN annotation_quality_score < 0.5 THEN '0.0 - 0.49 (Poor)'
        WHEN annotation_quality_score < 0.7 THEN '0.5 - 0.69 (Below Average)'
        WHEN annotation_quality_score < 0.8 THEN '0.7 - 0.79 (Average)'
        WHEN annotation_quality_score < 0.9 THEN '0.8 - 0.89 (Good)'
        ELSE '0.9 - 1.0 (Excellent)'
    END AS quality_range,
    COUNT(*) AS num_annotations,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage,
    ROUND(AVG(CASE WHEN error_flag = 1 THEN 1 ELSE 0 END) * 100, 2) AS error_rate_pct
FROM annotations
WHERE annotation_quality_score IS NOT NULL
GROUP BY 
    CASE 
        WHEN annotation_quality_score < 0.5 THEN '0.0 - 0.49 (Poor)'
        WHEN annotation_quality_score < 0.7 THEN '0.5 - 0.69 (Below Average)'
        WHEN annotation_quality_score < 0.8 THEN '0.7 - 0.79 (Average)'
        WHEN annotation_quality_score < 0.9 THEN '0.8 - 0.89 (Good)'
        ELSE '0.9 - 1.0 (Excellent)'
    END
ORDER BY quality_range;
```

### Query 25: Confidence vs Actual Performance
**Purpose**: Check if annotators' confidence correlates with actual quality

```sql
SELECT 
    CASE 
        WHEN confidence_score < 0.6 THEN 'Low Confidence (<0.6)'
        WHEN confidence_score < 0.8 THEN 'Medium Confidence (0.6-0.79)'
        ELSE 'High Confidence (0.8+)'
    END AS confidence_level,
    COUNT(*) AS total_annotations,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(confidence_score), 2) AS avg_confidence_score
FROM annotations
WHERE confidence_score IS NOT NULL
GROUP BY 
    CASE 
        WHEN confidence_score < 0.6 THEN 'Low Confidence (<0.6)'
        WHEN confidence_score < 0.8 THEN 'Medium Confidence (0.6-0.79)'
        ELSE 'High Confidence (0.8+)'
    END
ORDER BY confidence_level;
```

### Query 26: Review Coverage Analysis
**Purpose**: Understand what percentage of annotations are reviewed

```sql
SELECT 
    platform,
    project_name,
    COUNT(*) AS total_annotations,
    SUM(CASE WHEN is_reviewed THEN 1 ELSE 0 END) AS reviewed_annotations,
    SUM(CASE WHEN NOT is_reviewed THEN 1 ELSE 0 END) AS unreviewed_annotations,
    ROUND(AVG(CASE WHEN is_reviewed THEN 1 ELSE 0 END) * 100, 2) AS review_coverage_pct
FROM annotations
GROUP BY platform, project_name
ORDER BY review_coverage_pct ASC;
```

---

## Advanced Analytics Queries

### Query 27: Annotator Cohort Analysis
**Purpose**: Track performance improvement over time for new annotators

```sql
WITH annotator_performance_by_period AS (
    SELECT 
        annotator_id,
        platform,
        CASE 
            WHEN annotator_experience_days <= 30 THEN 'First Month'
            WHEN annotator_experience_days <= 90 THEN 'Months 2-3'
            WHEN annotator_experience_days <= 180 THEN 'Months 4-6'
            ELSE 'Beyond 6 Months'
        END AS experience_period,
        COUNT(*) AS annotations,
        ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct
    FROM annotations
    GROUP BY annotator_id, platform, 
        CASE 
            WHEN annotator_experience_days <= 30 THEN 'First Month'
            WHEN annotator_experience_days <= 90 THEN 'Months 2-3'
            WHEN annotator_experience_days <= 180 THEN 'Months 4-6'
            ELSE 'Beyond 6 Months'
        END
)

SELECT 
    platform,
    experience_period,
    COUNT(DISTINCT annotator_id) AS num_annotators,
    SUM(annotations) AS total_annotations,
    ROUND(AVG(error_rate_pct), 2) AS avg_error_rate_pct,
    ROUND(MIN(error_rate_pct), 2) AS best_error_rate,
    ROUND(MAX(error_rate_pct), 2) AS worst_error_rate
FROM annotator_performance_by_period
GROUP BY platform, experience_period
ORDER BY platform, 
    CASE experience_period
        WHEN 'First Month' THEN 1
        WHEN 'Months 2-3' THEN 2
        WHEN 'Months 4-6' THEN 3
        ELSE 4
    END;
```

### Query 28: Reviewer Workload and Quality
**Purpose**: Analyze reviewer performance and workload distribution

```sql
SELECT 
    reviewer_id,
    COUNT(*) AS annotations_reviewed,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score_given,
    SUM(error_flag) AS errors_flagged,
    ROUND(AVG(error_flag) * 100, 2) AS error_flag_rate_pct,
    COUNT(DISTINCT annotator_id) AS unique_annotators_reviewed,
    COUNT(DISTINCT DATE(submission_timestamp)) AS days_active
FROM annotations
WHERE is_reviewed = TRUE
GROUP BY reviewer_id
ORDER BY annotations_reviewed DESC;
```

### Query 29: Error Prediction Features
**Purpose**: Identify factors that predict errors (for ML model building)

```sql
SELECT 
    platform,
    training_completed,
    CASE 
        WHEN annotator_experience_days < 30 THEN 'Novice'
        WHEN annotator_experience_days < 90 THEN 'Intermediate'
        ELSE 'Experienced'
    END AS experience_level,
    time_of_day,
    CASE 
        WHEN time_spent_seconds < 30 THEN 'Very Fast'
        WHEN time_spent_seconds < 60 THEN 'Fast'
        WHEN time_spent_seconds < 120 THEN 'Normal'
        ELSE 'Slow'
    END AS speed_category,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct
FROM annotations
GROUP BY 
    platform, 
    training_completed, 
    CASE 
        WHEN annotator_experience_days < 30 THEN 'Novice'
        WHEN annotator_experience_days < 90 THEN 'Intermediate'
        ELSE 'Experienced'
    END,
    time_of_day,
    CASE 
        WHEN time_spent_seconds < 30 THEN 'Very Fast'
        WHEN time_spent_seconds < 60 THEN 'Fast'
        WHEN time_spent_seconds < 120 THEN 'Normal'
        ELSE 'Slow'
    END
HAVING COUNT(*) >= 20
ORDER BY error_rate_pct DESC;
```

### Query 30: ROI Analysis - Cost of Errors by Platform
**Purpose**: Estimate the impact of errors (assuming rework cost)

```sql
-- Assuming each annotation costs $0.10 and each error requires $0.20 rework
WITH platform_costs AS (
    SELECT 
        platform,
        COUNT(*) AS total_annotations,
        SUM(error_flag) AS total_errors,
        COUNT(*) * 0.10 AS annotation_cost,
        SUM(error_flag) * 0.20 AS rework_cost,
        (COUNT(*) * 0.10) + (SUM(error_flag) * 0.20) AS total_cost
    FROM annotations
    GROUP BY platform
)

SELECT 
    platform,
    total_annotations,
    total_errors,
    ROUND(annotation_cost, 2) AS annotation_cost_usd,
    ROUND(rework_cost, 2) AS rework_cost_usd,
    ROUND(total_cost, 2) AS total_cost_usd,
    ROUND(rework_cost / total_cost * 100, 2) AS rework_pct_of_total,
    ROUND(total_cost / total_annotations, 4) AS cost_per_annotation
FROM platform_costs
ORDER BY total_cost DESC;
```

---

## Summary Query - Executive Dashboard

### Query 31: Executive Summary Dashboard
**Purpose**: Single query for high-level executive overview

```sql
WITH platform_summary AS (
    SELECT 
        platform,
        COUNT(*) AS total_annotations,
        SUM(error_flag) AS total_errors,
        ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
        ROUND(AVG(annotation_quality_score), 2) AS avg_quality,
        ROUND(AVG(time_spent_seconds), 2) AS avg_time,
        COUNT(DISTINCT annotator_id) AS num_annotators,
        COUNT(DISTINCT project_name) AS num_projects
    FROM annotations
    GROUP BY platform
),
error_breakdown AS (
    SELECT 
        platform,
        error_type,
        COUNT(*) AS error_count
    FROM annotations
    WHERE error_flag = 1
    GROUP BY platform, error_type
)

SELECT 
    ps.platform,
    ps.total_annotations,
    ps.total_errors,
    ps.error_rate_pct,
    ps.avg_quality,
    ps.avg_time,
    ps.num_annotators,
    ps.num_projects,
    COALESCE(eb.error_count, 0) AS guideline_confusion_errors,
    ROUND(COALESCE(eb.error_count, 0) * 100.0 / NULLIF(ps.total_errors, 0), 2) AS pct_guideline_confusion
FROM platform_summary ps
LEFT JOIN error_breakdown eb ON ps.platform = eb.platform AND eb.error_type = 'guideline_confusion'
ORDER BY ps.platform;
```

---

## Query Index for Quick Reference

### Error Analysis:
- Query 4: Overall error rate
- Query 5: Error rate by platform ⭐
- Query 7: Error type by platform ⭐ (KEY INSIGHT)
- Query 10: Guideline confusion analysis ⭐ (KEY FINDING)

### Platform Comparison:
- Query 9: Comprehensive platform comparison ⭐
- Query 11: Project switching impact ⭐

### Performance:
- Query 12: Top performers
- Query 14: Experience vs errors ⭐
- Query 15: Training impact

### Task Analysis:
- Query 16: Error rate by task type ⭐
- Query 17: Task complexity

### Time Analysis:
- Query 19: Monthly trends
- Query 20: Time of day performance

### Advanced:
- Query 27: Cohort analysis ⭐
- Query 30: ROI analysis
- Query 31: Executive dashboard ⭐

**⭐ = Most important queries for your analysis**
