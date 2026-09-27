# MedCare Pharma

### Inventory Alerts + Demand Sensing for Pharmaceutical Supply Chains

MedCare Pharma is a supply-chain decision support system that combines **current inventory monitoring (E1)** with **30-day demand forecasting (P1)** to help planners make better replenishment decisions.

The system recommends one of three actions:

- **TRANSFER** – Move surplus stock from another warehouse.
- **REORDER** – Purchase only the remaining shortage.
- **NO ACTION** – No intervention required.

## Key Features

- 30-day demand forecasting using Chronos-2
- Inventory and stock-out risk monitoring
- Safety stock and lead-time analysis
- Batch-level expiry tracking and FEFO
- Cross-warehouse stock allocation
- Transfer and reorder recommendations
- Email alerts for high-risk items
- 30-day inventory simulation
- Interactive dashboard
- Automated testing using Pytest

## Tech Stack

- Python
- Pandas & NumPy
- Chronos-2
- PyTorch & Transformers
- SQLite
- Streamlit
- HTML, CSS & JavaScript
- Plotly
- Pytest
- GitHub Pages

## Project Flow

Demand + Inventory + Batch Data  
↓  
Database Layer  
↓  
Demand Forecast  
↓  
Risk Analysis  
↓  
Expiry & Allocation  
↓  
Decision Engine  
↓  
TRANSFER / REORDER / NO ACTION  
↓  
Dashboard & Alerts

## Project Structure

```text
medcare/
├── data/
├── database/
├── decision_engine/
├── integration/
├── ml/
├── notification_engine/
├── operations/
├── tests/
├── outputs/
├── main.py
├── streamlit_app.py
└── medcare_dashboard.html
