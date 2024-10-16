### Part 1. Data Exploration and analysis

**Q1) Data Quality Issues**
1. Irrelevant Data
    - Patient personal details such as DOB, Area code, should be removed as these columns do not support any calculation and purpose in data analysis.
    - Steps taken: Drop columns DOB, Area Code
    - Improvements: Implement data de-identification methods to remove patient details that might expose patient identity while retaining necessary data for analysis.
2. Inconsistent Formatting and Inaccurate Data
    - Columns 'AdmissionTime', 'SeparationTime', 'PharmacyCharged' have incorrect data types.
    - 'PharmacyCharged' has inconsistent data to other charges column, which will lead to calculation challenges. This column also has very large numbers that should be rechecked with data source, which may indicate inaccuracy of data.
    - Steps taken:
        -'AdmissionTime' and 'SeparationTime' converted to datetime
        -'PharmacyCharged' is in exponential terms - dropped for this analysis as the numbers are very large which does not make sense.
    - Improvements: 
        - Establish data transformation pipelines that include checks for data quality and type conversion as part of the data processing workflow.
        - Reach out to backend/engineering team to fix data exporting issue regarding values for 'PharmacyCharged' or if data was sourced from data warehouse, confirm that the values are right and if necessary seek advise to validate the values are right.

**Q2)** The feature created is the TotalCharges, which is the sum of all charges for each episode_id. When grouped by AR-DRG, this will provide insights on the total charges and form
the basis of activity-based Funding (ABF), where hospitals are paid based on the number
and complexity of cases they treat. It can also be used for predictive forecasting to make business decisions, such as determining which DRG areas require more resources or areas that are not significant enough to provide continuing service to reduce cost/expenses. This also gives an idea on budget predictability and financial planning to manage operational costs and plan for future services and needs.

### Part 2: Data Analysis and Visualisation
**Q3)** The DRG with the largest chargers in descending order is: DRG002, DRG003 and DRG001.

Using a time-series line graph, we can observe the top 3 DRG trends through the months from the earliest admission date (August 2022) to the latest admission date (July 2024). It is also useful to know which Care Types are causing the trends in the graph to know what drives these changes. We can also further analyse the Mode of Separations, and Discharge Intentions to extract more information on our findings.

Results:
1. DRG001 is likely linked to urgent conditions or acute care sevices, where majority patients are discharged home. Spikes in April might correspond to higher ED admissions due to seasonal illness or other acute outbreaks. The major spike in April 2023 can be observed in inpatient care types as well, which suggest possible incentives on post-pandemic elective surgery recovery works. 
2. DRG002 has a higher outpatient trend that indicates possible focus on managing chronic conditions or long-term care, this is also reflected with majority patients being discharged with palliative intentions. The significant drop in April 2024 suggests that it might be due to a reason that shifted the focus from DRG003 to DRG001.
3. DRG003 may involve frequent or regular procedures, though ED presentations for this DRG were on the higher range between Nov22 and peaked in Jun23. This indicates that this group could be tied to recurring conditions or planned procedures, where patients are often transferred to rehabilitation or home after discharge.

Fundamentally, it is important to understand the nature of the healthcare business to connect the dots on what is causing data trends and patterns. Some examples include changes in healthcare policies, Medicare benefits expansions, addition of medical centres/clinic, improvements in diagnostic toolings, etc. 

**Q4)**
```sql
SELECT 
    AdmissionMonth AS Period, 
    COUNT(episode_id) AS total_admissions, 
    COUNT(episode_id)/23 as average_admission 
    FROM df_1 
    GROUP BY AdmissionMonth 
    ORDER BY SUBSTR(Period, 4, 4), SUBSTR(Period, 1, 2)
```

**Q5)**
```sql
WITH RankedData AS 
    (SELECT 
        Sex, 
        PrincipalDiagnosis, 
        TotalCharges, 
        ROW_NUMBER() OVER (PARTITION BY Sex, PrincipalDiagnosis ORDER BY TotalCharges) AS row, 
        COUNT(*) OVER (PARTITION BY PrincipalDiagnosis, Sex) AS cnt 
    FROM df_1), 
    Percentile AS 
    (SELECT 
        Sex, 
        PrincipalDiagnosis, 
        MAX(CASE WHEN cnt = 1 THEN TotalCharges WHEN row = ROUND(cnt * 0.25) THEN TotalCharges END) AS percentile_25, 
        MAX(CASE WHEN cnt = 1 THEN TotalCharges WHEN row = ROUND(cnt * 0.50) THEN TotalCharges END) AS percentile_50, 
        MAX(CASE WHEN cnt = 1 THEN TotalCharges WHEN row = ROUND(cnt * 0.75) THEN TotalCharges END) AS percentile_75 
    FROM RankedData 
    GROUP BY PrincipalDiagnosis, Sex) 
    SELECT 
        Sex, 
        PrincipalDiagnosis, 
        percentile_25, 
        percentile_50, 
        percentile_75 
    FROM Percentile 
    ORDER BY PrincipalDiagnosis, Sex
```
The RankedData CTE table contains the row column to give a row number to each row grouped by Sex and Principal Diagnosis. Another column is created named cnt acts as the counter to count the total number of row in those groups. 

The Percentile CTE table manually calculates the percentiles 25, 50, 75, by using conditional statements. If there is only 1 record in the group, then the TotalCharges will be returned for all percentiles as that is the one and only record. If there is more than 1, then the percentile is calculated by using the rounded row number*percentile, which returns the TotalCharge corresponding to that row number. 

### Part 4: Strategic Insights and Recommendations
**Q6)** Two strategic insights:
1. Average admissions are highest during months May-July over the last 2 years. This aligns with predictable seasonal surge.It would be recommended to allocate additional resouces to accomodate for this increase in demand. Financial plannings can also be done for temporary expansions in service capcity where demand is most prominent. This helps Ramsay in scaling services that can lead to reduced wait times and optimizes resoruce utilization.
2. Reducing Unplanned Readmissions and Unplanned Theatre Visit. This depends on the targets and also whether there have been consistently high unplanned readmissions/theatre visit. There is notable frequency in Unplanned Readmissions in DRG003 for Inpatient Care Types, and Unplanned Threatre Visits in DRG002 for Inpatient Care Types. This helps improve patient care quality and reduce unnecessary costs and burden in theathre scheduling.

### Part 5: Model Development
**Q7) Predictive Modeling**
Model purpose: The purpose of the predictive model is to forecast future by utilising historical time-based DRG trends. This will allow Ramsay to anticipate fluctuations in resource demands and make informed decisions about staffing, budgeting, and facility management, and also help prepare for increased patient loads.
Model choice: As the data relies on historical data in a period of time, a time series model is recommended. We can use a machine learning model like LTSM, which is a type of recurrent nueral network that can capture long-term dependencies in sequential data. 
Preprocessing steps: This stage handles tasks like data preparation, feature engineering and normalization so that features are on similar scale for training.
Evaluation metrics: The evaluation metrics used for LTSM data are Root Mean Square Error (RMSE), Mean Abosolute Error (MAE), Mean Absolute Percentage Error (MAPE), R^2 score.


### Part 6: MLOps and Deployment
**Q8) Model deployment**
There are mainly three steps in model deployment:
1. Preparation of model - Training and testing of model.
2. Select platform to deploy model - This stage involves working with engineering teams to create a pipeline for model deployment to a platform and creating API endpoints for users to use the model by inputting data.  
3. Monitor and update model - over time the model will start to deterioriate if there are new patterns and behaviors. The maintenance of deployed models can be done by regularly assessing evaluation metrics to ensure that the model is performing as expected. Real-time monitoring can also be done to capture and log model predictions, input features and actual outcomes and alerts can be set up for performance anomalies. Thresholds for performance metrics can also be established to trigger alerts if the model starts to perfom below acceptable levels. A/B testing can also be used to compare perofmrnace of the deployment model against a control model for assessment. User feedbacks are also important in the monitoring of model performance and detection of possible improvements.
