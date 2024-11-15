# Forfun
import pandas as pd
import numpy as np
from scipy import stats
from statsmodels.tsa.stattools import adfuller, grangercausalitytests
from google.cloud.bigquery import Client
import warnings

warnings.filterwarnings("ignore")

class TimeSeriesFeatureEngineering:
    def __init__(self, project_id, table, date_cutoff, unique_id="all"):
        self.client = Client(project=project_id)
        self.table = table
        self.date_cutoff = date_cutoff
        self.unique_id = unique_id
        self.data = None
        self.transformed_data = None
        self.selected_features_dict = {}
    
    def load_data(self):
        query = f"""SELECT * FROM {self.table}"""
        self.data = self.client.query(query).to_dataframe()
        self.data = self.data[(self.data["unique_id"] == self.unique_id) & 
                              (self.data["date"] < self.date_cutoff)].reset_index(drop=True)
        print("Data loaded with shape:", self.data.shape)

    def select_columns(self, columns):
        self.data = self.data[columns].copy()
        self.data.set_index("date", inplace=True)
        self.data.sort_index(inplace=True)
        print("Columns selected with shape:", self.data.shape)

    def check_negatives(self, columns):
        negative_columns = [col for col in columns if (self.data[col] <= 0).any()]
        print("Columns with negative values:", negative_columns)
        return [col for col in columns if col not in negative_columns]

    def boxcox_transform(self, columns):
        transformed_data = self.data[columns].apply(lambda x: stats.boxcox(x)[0] if (x > 0).all() else x)
        self.transformed_data = transformed_data.dropna().copy()
        print("Box-Cox Transformation applied.")
    
    def check_stationarity(self, columns, signif=0.05):
        non_stationary_columns = []
        for column in columns:
            adf_result = adfuller(self.transformed_data[column], autolag="AIC")
            p_value = adf_result[1]
            if p_value > signif:
                non_stationary_columns.append(column)
        print("Non-stationary columns:", non_stationary_columns)
        return non_stationary_columns

    def apply_differencing(self, columns, order=2):
        for _ in range(order):
            self.transformed_data[columns] = self.transformed_data[columns].diff()
            self.transformed_data.dropna(inplace=True)
        print(f"{order}-order differencing applied.")
    
    def granger_causality_test(self, max_lag=12, signif=0.05):
        causality_df = pd.DataFrame(np.zeros((len(self.transformed_data.columns), len(self.transformed_data.columns))), 
                                    index=self.transformed_data.columns, columns=self.transformed_data.columns)
        
        for col in causality_df.columns:
            for row in causality_df.index:
                test_result = grangercausalitytests(self.transformed_data[[row, col]], max_lag, verbose=False)
                min_p_value = min([round(test_result[i+1][0]['ssr_chi2test'][1], 4) for i in range(max_lag)])
                causality_df.loc[row, col] = min_p_value

        causality_df = causality_df.applymap(lambda x: x < signif)
        print("Granger causality test completed.")
        return causality_df

    def find_correlated_features(self, df, threshold=0.9):
        corr_matrix = df.corr().abs()
        upper = corr_matrix.where(np.triu(np.ones(corr_matrix.shape), k=1).astype(bool))
        to_drop = [column for column in upper.columns if any(upper[column] > threshold)]
        return to_drop

    def feature_selection(self, causal_df, exclude_list, groups):
        for group in groups:
            significant_features = [f for f in causal_df[causal_df[group + "_x"] < 0.05][group + "_x"].index if f not in exclude_list]
            significant_features.append(group)
            self.selected_features_dict[group] = significant_features

    def run_pipeline(self, columns, exclude_list, groups):
        self.load_data()
        self.select_columns(columns)
        
        # Check negative values and apply Box-Cox transformation
        selected_columns = self.check_negatives(columns)
        self.boxcox_transform(selected_columns)

        # Stationarity and differencing
        non_stationary_columns = self.check_stationarity(selected_columns)
        if non_stationary_columns:
            self.apply_differencing(non_stationary_columns)

        # Granger Causality Test
        causal_df = self.granger_causality_test()
        
        # Feature Selection based on Granger causality and correlation
        self.feature_selection(causal_df, exclude_list, groups)
        
        print("Final selected features:", self.selected_features_dict)

# Define parameters
columns = ['date', 'unemployment_rate', 'inflation_value', 'us_interest_rate', 'consumer_sentiment', 'total_consumer_debt', 'total_consumer_debt_monthlyrate', 
           'total_group_cnt', 'total_member_cnt', 'bkt_1_M_member_perc', 'bkt_1_F_member_perc', 'bkt_2_M_member_perc', 'bkt_2_F_member_perc', 
           'bkt_3_M_member_perc', 'bkt_3_F_member_perc', 'bkt_4_M_member_perc', 'bkt_4_F_member_perc', 'gnrc_cnt', 'jcode_cnt', 'brnd_cnt', 
           'new_drug_cnt', 'jcode_allowed', 'jcode_paid', 'jcode_3k_allowed', 'jcode_3k_paid', 'jcode_top15_allowed', 'jcode_top15_paid', 
           'chronic_allowed', 'chronic_paid', 'acute_facility_cnt', 'bh_facility_cnt', 'facility_cnt', 'non_facility_cnt', 'non_acute_facility_cnt', 
           'r_facility_cnt', 'other_facility_cnt', 'primary_care_cnt', 'specialist_cnt', 'physician_cnt', 'nursing_cnt', 
           'healthcare_construction_spending', 'medical_care_cpi', 'std_arme_risk_score', 'std_arme_pmpm_pred', 'mean_arme_risk_score', 
           'mean_arme_pmpm_pred', 'pme_cnt', 'agesexfactor', 'ip_pmpm', 'op_pmpm', 'phy_pmpm', 'mh_pmpm', 'ip', 'phy', 'op', 'rx', 'med_pmpm', 
           'rx_pmpm', 'total_pmpm']
exclude_list = ["ip_y", "op_y", "phy_y", "rx_y", "med_pmpm_y", "rx_pmpm_y", "total_pmpm_y", "ip_pmpm_y", "op_pmpm_y", "phy_pmpm_y", "mh_pmpm_y"]
groups = ["ip", "op", "phy", "rx", "med_pmpm", "rx_pmpm", "total_pmpm", "ip_pmpm", "op_pmpm", "phy_pmpm", "mh_pmpm"]

# Initialize and run
project_id = "anbc-hcb-dev"
table = "pricing_arme_ds_hcb_dev.TMP_TM_ALL_FEATURES_TARGET_modeling_20241015"
date_cutoff = "2022-08-01"

ts_feature_engineering = TimeSeriesFeatureEngineering(project_id, table, date_cutoff)
ts_feature_engineering.run_pipeline(columns, exclude_list, groups)

# Final selected features dictionary
selected_features = ts_feature_engineering.selected_features_dict
pd.DataFrame(dict([(k, pd.Series(v)) for k, v in selected_features.items()]))
