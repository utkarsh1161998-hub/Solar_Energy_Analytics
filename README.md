import json
import logging
from datetime import datetime

# Setup basic logger
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

class SolarEdgeAnalyzer:
    def __init__(self, capacity_kw, efficiency_stc, temp_coeff):
        """
        Initialize the edge solar analysis node.
        :param capacity_kw: Nameplate DC capacity of the array in kW
        :param efficiency_stc: Panel efficiency at Standard Test Conditions (e.g., 0.20 for 20%)
        :param temp_coeff: Power temperature coefficient (e.g., -0.0038 for -0.38%/C)
        """
        self.capacity = capacity_kw
        self.eff_stc = efficiency_stc
        self.temp_coeff = temp_coeff  # Negative value indicating drop per degree C above 25C
        self.soiling_threshold = 0.88 # Trigger maintenance if performance index falls below 88%

    def calculate_performance_index(self, telemetry):
        """
        Computes the temperature-corrected performance ratio.
        """
        irradiance = telemetry["poa_irradiance_w_m2"]
        mod_temp = telemetry["module_temp_c"]
        actual_power_kw = telemetry["inverter_dc_power_kw"]

        if irradiance < 100:
            # Low light conditions (e.g., dawn/dusk/night); suppress false anomaly alerts
            return None

        # 1. Calculate theoretical expected power output considering temperature losses
        # STC Irradiance baseline is 1000 W/m2
        expected_power_no_temp = self.capacity * (irradiance / 1000.0)
        
        # Temperature delta relative to 25°C STC baseline
        temp_delta = mod_temp - 25.0
        thermal_derate_factor = 1.0 + (self.temp_coeff * temp_delta)
        
        expected_power_corrected = expected_power_no_temp * thermal_derate_factor

        # 2. Derive Performance Ratio (PR)
        performance_ratio = actual_power_kw / expected_power_corrected
        return round(performance_ratio, 3)

    def process_telemetry_frame(self, raw_frame_json):
        try:
            data = json.loads(raw_frame_json)
            pr = self.calculate_performance_index(data)
            
            if pr is None:
                return {"status": "IDLE", "reason": "Insufficient irradiance for accurate analysis."}

            result = {
                "timestamp": data["timestamp"],
                "calculated_performance_ratio": pr,
                "status": "NORMAL"
            }

            # Evaluation logic for alerting systems
            if pr < self.soiling_threshold:
                result["status"] = "ANOMALY_DETECTED"
                result["action_required"] = "Schedule Cleandown / Inspect Array Strings"
                logging.warning(f"ALERT: Performance index degraded to {pr * 100}% on string {data['string_id']}")
            else:
                logging.info(f"Asset Health Status: Healthy operational matrix (PR: {pr * 100}%)")

            return result

        except Exception as e:
            logging.error(f"Failed to process edge data packet: {str(e)}")
            return {"status": "ERROR", "message": str(e)}

# --- SIMULATION RUN ---
if __name__ == "__main__":
    # Specs for a commercial 100 kW solar array block
    analyzer = SolarEdgeAnalyzer(capacity_kw=100.0, efficiency_stc=0.20, temp_coeff=-0.0038)

    # Simulated field telemetry payload coming from an edge Modbus broker
    normal_daytime_payload = json.dumps({
        "timestamp": datetime.utcnow().isoformat(),
        "string_id": "block_01_row_A",
        "poa_irradiance_w_m2": 850.0,  # Clear bright sun
        "module_temp_c": 45.0,         # Warm panels running above STC baseline
        "inverter_dc_power_kw": 73.1    # Yielding power taking nominal thermal losses
    })

    soiled_array_payload = json.dumps({
        "timestamp": datetime.utcnow().isoformat(),
        "string_id": "block_01_row_A",
        "poa_irradiance_w_m2": 850.0,
        "module_temp_c": 45.0,
        "inverter_dc_power_kw": 58.2    # Power drop caused by dust accumulation
    })

    print("--- Frame 1: Nominal Conditions ---")
    print(analyzer.process_telemetry_frame(normal_daytime_payload))

    print("\n--- Frame 2: Soiled / Shaded Degradation Conditions ---")
    print(analyzer.process_telemetry_frame(soiled_array_payload))
