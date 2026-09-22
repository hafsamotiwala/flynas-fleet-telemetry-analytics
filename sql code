-- ----------------------------------------------------------------------------
-- FORENSIC QUERY 01: HUB GROUND-BLEED & CASH DESTRUCTION AUDIT
-- ----------------------------------------------------------------------------
SELECT 
    f.origin AS departure_hub,
    COUNT(f.tail_number) AS total_monitored_flights,
    ROUND(AVG(f.taxi_out), 2) AS avg_taxi_out_minutes,
    -- (Actual Taxi Time - 4 Min Buffer) * 60 Sec * ICAO Idle Flow Rate * 2 Engines
    ROUND(SUM((f.taxi_out - 4) * 60 * e.fuel_flow_idle_kg_sec * 2), 2) AS total_wasted_fuel_kg,
    -- Financial Loss assuming $1.00 USD per kg of jet fuel
    ROUND(SUM((f.taxi_out - 4) * 60 * e.fuel_flow_idle_kg_sec * 2 * 1.00), 2) AS total_financial_loss_usd
FROM fact_flights f
JOIN dim_aircraft a ON f.tail_number = a.tail_number
JOIN dim_engine e ON a.engine_identification = e.engine_identification
WHERE f.reporting_airline = 'XY' 
  AND f.taxi_out > 4 -- Excludes flights that departed immediately within the buffer
GROUP BY f.origin
ORDER BY total_financial_loss_usd DESC;

-- ----------------------------------------------------------------------------
-- FORENSIC QUERY 02: MULTI-LEG DELAY CASCADE ENGINE (CORRECTED)
-- ----------------------------------------------------------------------------
WITH ranked_daily_schedule AS (
    SELECT 
        flight_date,
        tail_number,
        origin,
        dest,
        carrier_delay AS leg_1_carrier_delay,
        -- Fetching the operational latency of the sequential leg using relational stream layout
        LEAD(origin, 1) OVER (PARTITION BY tail_number, flight_date ORDER BY taxi_out DESC) AS leg_2_origin,
        LEAD(dest, 1) OVER (PARTITION BY tail_number, flight_date ORDER BY taxi_out DESC) AS leg_2_dest,
        LEAD(carrier_delay, 1) OVER (PARTITION BY tail_number, flight_date ORDER BY taxi_out DESC) AS leg_2_carrier_delay
    FROM fact_flights
    WHERE reporting_airline = 'XY'
)
SELECT 
    origin AS initial_origin,
    dest AS initial_dest,
    leg_2_dest AS sequential_dest,
    COUNT(*) AS total_monitored_cascades,
    ROUND(AVG(leg_1_carrier_delay), 2) AS avg_leg_1_delay_mins,
    ROUND(AVG(leg_2_carrier_delay), 2) AS avg_cascaded_leg_2_delay_mins,
    -- Quantifying the delay amplification factor across sequential routes
    ROUND(AVG(leg_2_carrier_delay) - AVG(leg_1_carrier_delay), 2) AS net_delay_amplification_mins
FROM ranked_daily_schedule
WHERE leg_2_origin IS NOT NULL 
  AND leg_1_carrier_delay > 0 
GROUP BY origin, dest, leg_2_dest
HAVING total_monitored_cascades > 10
ORDER BY net_delay_amplification_mins DESC;

-- ----------------------------------------------------------------------------
-- FORENSIC QUERY 03: PERISHABLE CATERING MARGIN PARADOX RECLAMATION
-- ----------------------------------------------------------------------------
SELECT 
    origin AS departure_hub,
    COUNT(*) AS total_short_haul_flights,
    -- Simulating baseline fresh meals loaded vs wasted due to short air times
    ROUND(SUM(air_time * 5.50), 2) AS estimated_fresh_inventory_cost_usd,
    -- Simulating a 54% wastage benchmark rate on perishable short hops
    ROUND(SUM(air_time * 5.50 * 0.54), 2) AS fresh_spoilage_cash_bleed_usd,
    -- Reclaiming capital assuming 100% rollover utility of shelf-stable items
    ROUND(SUM(air_time * 5.50 * 0.54 * 0.85), 2) AS reclaimed_margin_usd
FROM fact_flights
WHERE reporting_airline = 'XY'
  AND air_time < 90 -- Filters for short-haul tracks under 1.5 hours
GROUP BY origin
ORDER BY reclaimed_margin_usd DESC;

-- ----------------------------------------------------------------------------
-- FORENSIC QUERY 04: FLEET MAINTENANCE AOG RISK ALERT
-- ----------------------------------------------------------------------------
SELECT 
    f.tail_number,
    a.engine_model,
    COUNT(f.flight_id) AS total_accumulated_flight_cycles,
    ROUND(SUM(f.air_time), 2) AS total_airframe_hours,
    -- Flagging high-wear thresholds (e.g., past 25,000 air minutes)
    CASE 
        WHEN SUM(f.air_time) > 25000 THEN 'CRITICAL RISK: Immediate Hangar Audit Required'
        WHEN SUM(f.air_time) BETWEEN 15000 AND 25000 THEN 'WARNING: Schedule ROP Part Trigger'
        ELSE 'OPTIMAL: Routine Line Maintenance'
    END AS fleet_maintenance_status,
    -- Financial Exposure calculation: If status is Critical, model a potential 3-day AOG grounding loss ($75,000)
    CASE 
        WHEN SUM(f.air_time) > 25000 THEN ROUND(3 * 25000.00, 2)
        ELSE 0.00
    END AS potential_aog_financial_exposure_usd
FROM (
    -- Generating a temporary row index to act as a pseudo-flight_id since our DDL lacked one
    SELECT *, ROW_NUMBER() OVER (ORDER BY flight_date) AS flight_id 
    FROM fact_flights 
    WHERE reporting_airline = 'XY'
) f
JOIN (
    SELECT ta.tail_number, en.engine_identification AS engine_model
    FROM dim_aircraft ta
    JOIN dim_engine en ON ta.engine_identification = en.engine_identification
) a ON f.tail_number = a.tail_number
GROUP BY f.tail_number, a.engine_model
ORDER BY total_airframe_hours DESC;
