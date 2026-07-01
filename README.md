import os
import random
import sqlite3
from datetime import datetime, timedelta
import matplotlib.pyplot as plt
import pandas as pd

def generate_logistics_data(num_records=5000):
    transit_modes = ["Air Freight", "Ocean Cargo", "Ground Trucking"]
    status_options = ["Delivered", "In Transit", "Delayed"]
    start_date = datetime(2026, 1, 1)
    
    data = []
    for i in range(num_records):
        trip_id = f"TRIP-{100000 + i}"
        mode = random.choice(transit_modes)
        
        if mode == "Air Freight":
            duration = round(random.uniform(12.0, 48.0), 2)
        elif mode == "Ocean Cargo":
            duration = round(random.uniform(120.0, 360.0), 2)
        else:
            duration = round(random.uniform(24.0, 96.0), 2)
            
        if random.random() < 0.10:
            duration = None
            
        days_offset = random.randint(0, 180)
        ship_date = (start_date + timedelta(days=days_offset)).strftime("%Y-%m-%d")
        distance_miles = random.randint(150, 4500)
        cost_per_mile = round(random.uniform(1.50, 4.75), 2)
        status = random.choice(status_options)
        
        data.append((trip_id, mode, ship_date, distance_miles, cost_per_mile, duration, status))
    return data

def main():
    raw_data = generate_logistics_data()
    columns = [
        "trip_id", "transit_mode", "ship_date", "distance_miles", 
        "cost_per_mile", "duration_hours", "status"
    ]
    df = pd.DataFrame(raw_data, columns=columns)
    
    median_duration = df["duration_hours"].median()
    df["duration_hours"] = df["duration_hours"].fillna(median_duration)
    
    df["total_operational_cost"] = df["distance_miles"] * df["cost_per_mile"]
    
    db_name = "logistics_warehouse.db"
    conn = sqlite3.connect(db_name)
    cursor = conn.cursor()
    
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS shipments (
            trip_id TEXT PRIMARY KEY,
            transit_mode TEXT,
            ship_date TEXT,
            distance_miles INTEGER,
            cost_per_mile REAL,
            duration_hours REAL,
            status TEXT,
            total_operational_cost REAL
        )
    """)
    conn.commit()
    
    df.to_sql("shipments", conn, if_exists="replace", index=False)
    conn.close()
    
    executive_matrix = df.groupby("transit_mode")[["total_operational_cost", "duration_hours"]].agg(
        {"total_operational_cost": "sum", "duration_hours": "mean"}
    ).rename(
        columns={
            "total_operational_cost": "Total_Cost_INR",
            "duration_hours": "Average_Duration_Hrs"
        }
    )
    
    executive_matrix.to_csv("transit_efficiency_report.csv")
    
    plt.figure(figsize=(8, 5))
    plt.bar(
        executive_matrix.index,
        executive_matrix["Total_Cost_INR"] / 1000,
        color=["#2b5c8f", "#4682b4", "#6baed6"],
        edgecolor="black"
    )
    plt.title("Total Operational Cost Distribution by Transit Mode", fontsize=12, fontweight="bold")
    plt.xlabel("Transit Infrastructure Classification", fontsize=10)
    plt.ylabel("Total Expenditure (Thousands)", fontsize=10)
    plt.grid(axis="y", linestyle="--", alpha=0.7)
    plt.tight_layout()
    plt.savefig("transit_cost_chart.png", dpi=300)
    plt.close()

if __name__ == "__main__":
    main()
