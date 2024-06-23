# EvaluateManufacturingProcess
# Manufacturing Process Monitoring

## Overview

This project supports a team that wants to improve how they monitor and control a manufacturing process using statistical process control (SPC). SPC is an established strategy that uses data to determine whether the process works well. Processes are only adjusted if measurements fall outside of an acceptable range defined by control limits.

## Goal

The goal is to implement a more methodical approach to monitoring the manufacturing process, ensuring high-quality products by identifying any points that fall outside of acceptable control limits.

## Data

The data is available in the `manufacturing_parts` table which has the following fields:

- `item_no`: the item number
- `length`: the length of the item made
- `width`: the width of the item made
- `height`: the height of the item made
- `operator`: the operating machine

## Methodology

Using SQL window functions and nested queries, we will analyze historical manufacturing data to define the acceptable range and identify any points in the process that fall outside of the range and therefore require adjustments.

### Control Limits

- **Upper Control Limit (UCL)**: \( \text{avg_height} + 3 \times \left(\frac{\text{stddev_height}}{\sqrt{5}}\right) \)
- **Lower Control Limit (LCL)**: \( \text{avg_height} - 3 \times \left(\frac{\text{stddev_height}}{\sqrt{5}}\right) \)

### Query Description

The SQL script performs the following steps:

1. **Calculate Rolling Statistics**:
   - Calculate the rolling averages and standard deviations for each operator.

2. **Filter Rows with Sufficient Data**:
   - Filter rows where `ROW_NUMBER >= 5` to ensure there are enough data points.

3. **Compute Control Limits and Flag Alerts**:
   - Compute the Upper Control Limit (UCL) and Lower Control Limit (LCL).
   - Use a `CASE` statement to flag if the height is within the control limits.

### Final Query

```sql
SELECT 
    b.*, 
    CASE 
        WHEN b.height NOT BETWEEN b.lcl AND b.ucl THEN TRUE 
        ELSE FALSE 
    END AS alert
FROM (
    SELECT 
        A.operator,
        A.Row_number,
        A.height,
        A.avg_height,
        A.stddev_height,
        A.avg_height + 3 * (A.stddev_height / SQRT(5)) AS ucl,
        A.avg_height - 3 * (A.stddev_height / SQRT(5)) AS lcl
    FROM (
        SELECT 
            operator,
            ROW_NUMBER() OVER w AS Row_number, 
            height, 
            AVG(height) OVER w AS avg_height, 
            STDDEV(height) OVER w AS stddev_height
        FROM public.manufacturing_parts
        WINDOW w AS (
            PARTITION BY operator 
            ORDER BY item_no 
            ROWS BETWEEN 4 PRECEDING AND CURRENT ROW
        )
    ) AS A 
    WHERE A.Row_number >= 5
) AS b
ORDER BY b.operator, b.Row_number;
