# Data-Annotation-Error-analysis
A comprehensive data analysis comparing two annotation platform operational models and their impact on data quality.
**EXECUTIVE SUMMARY**
This project analyzes 100k annotation records from two data annotation platforms operating under different workflow models during 2024. The analysis reveals significant quality differences driven by operational design choices.

**KEY FINDINGS**
Platform B's single-project lock model reduces error rates by 42.5% compared to Platform A's flexible multi-project model (10.69% vs 18.57%).

**Primary Driver**
Guideline confusion from project switching accounts for 35.3% of Platform A's errors and 85% of the quality gap between platforms.

**Questions Answered**
1. Which platform model produces higher quality? → Platform B (single-project lock)
2. What drives annotation errors? → Project switching (35%), task ambiguity (20%), inexperience (25%)
3. How does experience affect quality? → New annotators have 87% higher error rates
4. Which tasks are hardest? → emotion_detection (23.45%) and organization_entity (21.78%)
5. Does training help? → Yes, but workflow design matters more (1.5pp improvement vs 8pp from project lock)
6. What's the business impact? → Platform A spends 13.2% more per annotation due to rework

find full documentation : [Documentation](https://drive.google.com/file/d/1AiYvTU74TN__JZ3t_dxyFIEpwjBh_Zh4/view?usp=sharing)




