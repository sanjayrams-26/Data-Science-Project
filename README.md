# Automated API-Based Inventory Auditing and Stockout Risk Forecasting
cd src; python run_pipeline.py   (starts the local mock API, extracts with Requests, cleans/audits/forecasts with Pandas)
python plots.py                  (creates figures in outputs/figures)
To use a real ERP/WMS: change base URL + API key in run_pipeline.py and map the field names.
