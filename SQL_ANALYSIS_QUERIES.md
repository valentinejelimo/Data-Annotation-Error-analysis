--Dataset Overview
```
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
FROM annotation;
```

---Platform Distribution
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

---Project Distribution
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

---Overall Error Rate
```sql
SELECT 
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    COUNT(*) - SUM(error_flag) AS total_correct,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage
FROM annotation;
```

---error rate between platform A and B
```sql
SELECT 
    platform,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    COUNT(*) - SUM(error_flag) AS total_correct,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotation
GROUP BY platform
ORDER BY error_rate_percentage DESC;
```

--- types of errors that are most common

```sql
SELECT 
    error_type,
    COUNT(*) AS occurrences,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS percentage_of_total
FROM annotation
WHERE error_flag = 1 
GROUP BY error_type
ORDER BY occurrences DESC;
```
---Identify platform-specific error patterns

```sql
SELECT 
    platform,
    error_type,
    COUNT(*) AS error_count,
    ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(PARTITION BY platform), 2) AS pct_of_platform_errors
FROM annotation
WHERE error_flag = 1
GROUP BY platform, error_type
ORDER BY platform, error_count DESC;
```

---Error Rate by Project
```sql
SELECT 
    platform,
    project_name,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_percentage,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score
FROM annotation
GROUP BY platform, project_name
ORDER BY error_rate_percentage DESC;
```

---

---Platform Performance Comparison
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
FROM annotation
GROUP BY platform
ORDER BY platform;
```

---Guideline Confusion Analysis
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
FROM annotation
GROUP BY platform
ORDER BY guideline_confusion_errors DESC;
```

---Project Switching Impact
```sql
WITH annotator_project_counts AS (
    SELECT 
        annotator_id,
        COUNT(DISTINCT project_name) AS projects_worked
    FROM annotation
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
Top Performing Annotators
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
FROM annotation
GROUP BY annotator_id, platform
HAVING COUNT(*) >= 50 
ORDER BY error_rate_pct ASC, total_annotations DESC
LIMIT 20;
```

---Worst Performing Annotators
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
FROM annotation
WHERE error_flag = 1
GROUP BY annotator_id, platform, training_completed
HAVING COUNT(*) >= 50
ORDER BY error_rate_pct DESC
LIMIT 20;
```

---Experience vs Error Rate
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
FROM annotation
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

---Training Impact
```sql
SELECT 
    platform,
    training_completed,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    COUNT(DISTINCT annotator_id) AS num_annotators
FROM annotation
GROUP BY platform, training_completed
ORDER BY platform, training_completed;
```

---

---Error Rate by Task Type
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

---Task Complexity Analysis

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
FROM annotation
GROUP BY task_type
ORDER BY avg_time_seconds DESC;
```
---Time of Day Performance
```sql
SELECT 
    time_of_day,
    COUNT(*) AS total_annotations,
    SUM(error_flag) AS total_errors,
    ROUND(AVG(error_flag) * 100, 2) AS error_rate_pct,
    ROUND(AVG(annotation_quality_score), 2) AS avg_quality_score,
    ROUND(AVG(time_spent_seconds), 2) AS avg_time_seconds,
    COUNT(DISTINCT annotator_id) AS num_annotators
FROM annotation
GROUP BY time_of_day
ORDER BY 
    CASE time_of_day
        WHEN 'Morning' THEN 1
        WHEN 'Afternoon' THEN 2
        WHEN 'Evening' THEN 3
        WHEN 'Night' THEN 4
    END;
```
---Quality Score Distribution
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
FROM annotation
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

--- Confidence vs Actual Performance
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
FROM annotation
WHERE confidence_score IS NOT NULL
GROUP BY 
    CASE 
        WHEN confidence_score < 0.6 THEN 'Low Confidence (<0.6)'
        WHEN confidence_score < 0.8 THEN 'Medium Confidence (0.6-0.79)'
        ELSE 'High Confidence (0.8+)'
    END
ORDER BY confidence_level;
```
---ROI Analysis - Cost of Errors by Platform
-- Assuming each annotation costs $0.10 and each error requires $0.20 rework
```sql
WITH platform_costs AS (
    SELECT 
        platform,
        COUNT(*) AS total_annotations,
        SUM(error_flag) AS total_errors,
        COUNT(*) * 0.10 AS annotation_cost,
        SUM(error_flag) * 0.20 AS rework_cost,
        (COUNT(*) * 0.10) + (SUM(error_flag) * 0.20) AS total_cost
    FROM annotation
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
