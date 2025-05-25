from http.server import BaseHTTPRequestHandler, HTTPServer
from typing import Any
from urllib.parse import parse_qs
import json
import threading
import time
import logging
import sys
from uart_comm import UARTCommunicator

# Configure logging to print to both file and console
logging.basicConfig(
    level=logging.INFO,
    format='%(message)s',  # Remove timestamp and level
    handlers=[
        logging.FileHandler('backend.log', mode='w'),  # Clear file on start
        logging.StreamHandler(sys.stdout)  # Print to console
    ]
)
logger = logging.getLogger(__name__)

class Config:
    def __init__(self):
        # Sensor values
        self.sensors = {
            "MainTankWaterLevel": 0.0,
            "HeadGP1WaterLevel": 0.0,
            "HeadGP2WaterLevel": 0.0,
            "MainTankTemp": 0.0,
            "HeadGP1TopTemp": 0.0,
            "HeadGP1BottomTemp": 0.0,
            "HeadGP2TopTemp": 0.0,
            "HeadGP2BottomTemp": 0.0,
            "Pressure": 0.0,
            "MainTankWaterFlow": 0.0,
            "HeadGP1WaterFlow": 0.0,
            "HeadGP2WaterFlow": 0.0,
            "Current": 0.0,
            "Voltage": 0.0
        }
        
        # Group Head 1 Configuration
        self.gh1_config = {
            "temperature": 91.0,
            "extraction_volume": 0,
            "extraction_time": 20,
            "pre_infusion": 0,
            "purge": 0,
            "backflush": False
        }
        
        # Group Head 2 Configuration
        self.gh2_config = {
                "temperature": 92.0,
            "extraction_volume": 0,
                "extraction_time": 20,
            "pre_infusion": 0,
            "purge": 0,
            "backflush": False
        }
        
        # System states
        self.FLOWGPH1CGF = 0.0
        self.FLOWGPH2CGF = 0.0
        self.tempMainTankFlag = True
        self.tempHeadGP1Flag = True
        self.tempHeadGP2Flag = True
        self.enableHeadGP1 = True
        self.enableHeadGP2 = True
        self.enableMainTank = True
        self.Pressure1 = 0.0
        self.Pressure2 = 0.0
        
        # UI settings
        self.HGP1FlowVolume = 0
        self.HGP2FlowVolume = 0
        self.HGP1PreInfusion = 0
        self.HGP2PreInfusion = 0
        self.HGP1ExtractionTime = 20
        self.HGP2ExtractionTime = 20
        self.tempMainTankSetPoint = 110.0
        self.tempHeadGP1SetPoint = 114.0
        self.tempHeadGP2SetPoint = 92.0
        
        # Additional features
        self.tempCupFlag = False
        self.cup = 0
        self.baristaLight = False
        self.light = 0
        self.ecomode = 0
        self.dischargeMode = 0
        
        # System states
        self.mainTankState = 1
        self.HGP1State = 1
        self.HGP2State = 1
        self.HGP1ACTIVE = 0
        self.HGP2ACTIVE = 0
        self.HGP12MFlag = 4
        self.sebar = 0
        self.HGPCheckStatus = False
        self.backflush1 = 4
        self.backflush2 = 4

        # Main ampere configuration
        self.mainAmpereConfig = {
            "temperature": 125.0,
            "pressure": 9.0
        }
        
        # Pressure configuration
        self.pressureConfig = {
            "pressure": 9.0,
            "max_pressure": 12.0,
            "min_pressure": 0.0
        }
        
        print("Configuration initialized")

# Create global config instance
config = Config()
uart = UARTCommunicator()
try:
    uart.start()
except Exception as e:
    print(f"UART not started: {e}")

main_boiler_state = 0  # 0 = off, 1 = on

# State tracking for UART data
last_gh1_data = None
last_gh2_data = None
last_main_data = None

gh1_button_state = 0  # 0 = off, 1 = on
gh2_button_state = 0  # 0 = off, 1 = on

# Helper to build and send GH1/GH2 string
def send_gh_uart(flag, cfg):
    temp = int(round(cfg['temperature']))
    ext_vol = int(round(cfg['extraction_volume']))
    ext_time = int(round(cfg['extraction_time']))
    purge = int(round(cfg['purge']))
    preinf_state = 1 if cfg['pre_infusion'] > 0 else 0
    preinf_time = int(round(cfg['pre_infusion']))
    backflush = 1 if cfg.get('backflush', False) else 0
    s = f"{flag};{temp};{ext_vol};{ext_time};{purge};{preinf_state};{preinf_time};{backflush}"
    uart.send_string(s)

# Helper to build and send main/buttons/pressure string
def send_main_uart():
    pressure = int(round(config.pressureConfig['pressure']))
    main_temp = int(round(config.mainAmpereConfig['temperature']))
    s = f"3;{main_boiler_state};{gh1_button_state};{gh2_button_state};{pressure};{main_temp}"
    uart.send_string(s)

def send_uart_on_mainboiler_change():
    uart_string = f"1;{main_boiler_state}"
    try:
        if hasattr(uart, 'ser') and uart.ser and uart.ser.is_open:
            uart.ser.write((uart_string + '\n').encode())
            print(f"UART SENT: {uart_string}")
        else:
            print(f"UART (not open): {uart_string}")
    except Exception as e:
        print(f"UART send error: {e}")

class RequestHandler(BaseHTTPRequestHandler):
    def log_message(self, format, *args):
        # Don't log HTTP requests
        pass
    
    def do_GET(self):
        if self.path == '/getmainstatus':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            status_data = {
                "main_temperature": {
                    "value": config.tempMainTankSetPoint,
                    "unit": "°C"
                },
                "gh1": {
                    "temperature": {
                        "value": config.tempHeadGP1SetPoint,
                        "unit": "°C"
                    },
                    "pressure": {
                        "value": config.Pressure1,
                        "unit": "bar"
                    }
                },
                "gh2": {
                    "temperature": {
                        "value": config.tempHeadGP2SetPoint,
                        "unit": "°C"
                    },
                    "pressure": {
                        "value": config.Pressure2,
                        "unit": "bar"
                    }
                }
            }
            
            self.wfile.write(json.dumps(status_data).encode())
            
        elif self.path == '/getdata':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            data = config.sensors.copy()
            
            # Add additional data
            data["HeadGP1WaterFlow"] = config.FLOWGPH2CGF
            data["HeadGP2WaterFlow"] = config.FLOWGPH1CGF
            data["HGP1ACTIVE"] = config.HGP1ACTIVE
            data["HGP2ACTIVE"] = config.HGP2ACTIVE
            data["mainTankState"] = config.mainTankState
            data["HGP1State"] = config.HGP1State
            data["HGP2State"] = config.HGP2State
            
            # Handle pressure data
            if config.sebar == 0:
                data["PressureGPH1"] = config.Pressure1
                data["PressureGPH2"] = config.Pressure2
            else:
                data["PressureGPH1"] = "3"
                data["PressureGPH2"] = "3"
            
            data["timeHGP1"] = config.timeHGP1
            data["timeHGP2"] = config.timeHGP2
            data["HeadGP1TopTemp"] = str(config.sensors["HeadGP1TopTemp"])
            data["HeadGP2TopTemp"] = str(config.sensors["HeadGP2TopTemp"])
            
            self.wfile.write(json.dumps(data).encode())
            
        elif self.path == '/geterror':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            data = {
                "HeadGroup1TemperatureStatus": config.tempHeadGP1Flag,
                "HeadGroup2TemperatureStatus": config.tempHeadGP2Flag,
                "MainTankTemperatureStatus": config.tempMainTankFlag
            }
            
            self.wfile.write(json.dumps(data).encode())
            
        elif self.path == '/getgauge':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            gauge_data = {
                "pressure": {
                    "value": config.Pressure1,
                    "min": 0,
                    "max": 12,
                    "unit": "bar"
                },
                "temperature": {
                    "value": config.tempMainTankSetPoint,
                    "min": 0,
                    "max": 120,
                    "unit": "°C"
                },
                "flow": {
                    "value": config.FLOWGPH1CGF,
                    "min": 0,
                    "max": 5,
                    "unit": "L/min"
                },
                "water_level": {
                    "value": config.sensors["MainTankWaterLevel"],
                    "min": 0,
                    "max": 100,
                    "unit": "%"
                }
            }
            
            self.wfile.write(json.dumps(gauge_data).encode())

        elif self.path == '/getghconfig':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            gh_config = {
                "gh1": config.gh1_config,
                "gh2": config.gh2_config
            }
            
            self.wfile.write(json.dumps(gh_config).encode())

        elif self.path == '/getmainconfig':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            main_config = config.mainAmpereConfig.copy()
            
            self.wfile.write(json.dumps(main_config).encode())
            
        elif self.path == '/getpressureconfig':
            self.send_response(200)
            self.send_header('Content-type', 'application/json')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            
            pressure_config = config.pressureConfig.copy()
            
            self.wfile.write(json.dumps(pressure_config).encode())
            
        else:
            self.send_response(404)
            self.send_header('Content-type', 'text/html')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            self.wfile.write(b'404 Not Found')

    def do_OPTIONS(self):
        self.send_response(200)
        self.send_header('Access-Control-Allow-Origin', '*')
        self.send_header('Access-Control-Allow-Methods', 'POST, GET, OPTIONS')
        self.send_header('Access-Control-Allow-Headers', 'Content-Type')
        self.end_headers()
        
    def do_POST(self):
        global last_main_data, last_gh1_data, last_gh2_data, main_boiler_state, gh1_button_state, gh2_button_state
        try:
            content_length = int(self.headers['Content-Length'])
            post_data = self.rfile.read(content_length).decode('utf-8')
            params = json.loads(post_data)
            
            if self.path == '/setpressureconfig':
                # Log the received request
                print("\nReceived POST request to /setpressureconfig")
                print("Request data:", json.dumps(params, indent=2))
                
                # Get the new configuration
                new_config = params.get('config', {})
                
                # Update pressure configuration
                config.pressureConfig.update({
                    "pressure": float(new_config.get('pressure', config.pressureConfig['pressure'])),
                    "max_pressure": float(new_config.get('max_pressure', config.pressureConfig['max_pressure'])),
                    "min_pressure": float(new_config.get('min_pressure', config.pressureConfig['min_pressure']))
                })
                
                # Log the updated configuration
                print("\nUpdated Pressure Configuration:")
                print("--------------------------------")
                print(json.dumps(config.pressureConfig, indent=2))
                print("--------------------------------\n")
                
                new_main = [main_boiler_state, gh1_button_state, gh2_button_state, int(round(config.pressureConfig['pressure'])), int(round(config.mainAmpereConfig['temperature']))]
                if last_main_data != new_main:
                    send_main_uart()
                    last_main_data = new_main
            
            elif self.path == '/setmainconfig':
                # Log the received request
                print("\nReceived POST request to /setmainconfig")
                print("Request data:", json.dumps(params, indent=2))
                
                # Get the new configuration
                new_config = params.get('config', {})
                
                # Update main configuration
                config.mainAmpereConfig.update({
                    "temperature": float(new_config.get('temperature', config.mainAmpereConfig['temperature']))
                })
                
                # Log the updated configuration
                print("\nUpdated Main Configuration:")
                print("--------------------------------")
                print(json.dumps({
                    "temperature": config.mainAmpereConfig['temperature']
                }, indent=2))
                print("--------------------------------\n")
                
                new_main = [main_boiler_state, gh1_button_state, gh2_button_state, int(round(config.pressureConfig['pressure'])), int(round(config.mainAmpereConfig['temperature']))]
                if last_main_data != new_main:
                    send_main_uart()
                    last_main_data = new_main
            
            elif self.path == '/saveghconfig':
                print("\nReceived POST request to /saveghconfig")
                print("Request data:", json.dumps(params, indent=2))
                new_config = params.get('config', {})
                gh_id = params.get('gh_id', 'ghundefined')
                # Update Group Head 1
                if gh_id == 'gh1':
                    config.gh1_config.update({
                        "temperature": float(new_config.get('temperature', config.gh1_config['temperature'])),
                        "extraction_volume": int(new_config.get('volume', config.gh1_config['extraction_volume'])),
                        "extraction_time": int(new_config.get('extraction_time', config.gh1_config['extraction_time'])),
                        "pre_infusion": int(new_config.get('pre_infusion', {}).get('time', config.gh1_config['pre_infusion'])),
                        "purge": int(new_config.get('purge', config.gh1_config['purge'])),
                        "backflush": bool(new_config.get('backflush', config.gh1_config['backflush']))
                    })
                    new_gh1 = [
                        int(round(config.gh1_config['temperature'])),
                        int(round(config.gh1_config['extraction_volume'])),
                        int(round(config.gh1_config['extraction_time'])),
                        int(round(config.gh1_config['purge'])),
                        1 if config.gh1_config['pre_infusion'] > 0 else 0,
                        int(round(config.gh1_config['pre_infusion'])),
                        1 if config.gh1_config.get('backflush', False) else 0
                    ]
                    if last_gh1_data != new_gh1:
                        send_gh_uart(1, config.gh1_config)
                        last_gh1_data = new_gh1
                # Update Group Head 2
                elif gh_id == 'gh2':
                    config.gh2_config.update({
                        "temperature": float(new_config.get('temperature', config.gh2_config['temperature'])),
                        "extraction_volume": int(new_config.get('volume', config.gh2_config['extraction_volume'])),
                        "extraction_time": int(new_config.get('extraction_time', config.gh2_config['extraction_time'])),
                        "pre_infusion": int(new_config.get('pre_infusion', {}).get('time', config.gh2_config['pre_infusion'])),
                        "purge": int(new_config.get('purge', config.gh2_config['purge'])),
                        "backflush": bool(new_config.get('backflush', config.gh2_config['backflush']))
                    })
                    new_gh2 = [
                        int(round(config.gh2_config['temperature'])),
                        int(round(config.gh2_config['extraction_volume'])),
                        int(round(config.gh2_config['extraction_time'])),
                        int(round(config.gh2_config['purge'])),
                        1 if config.gh2_config['pre_infusion'] > 0 else 0,
                        int(round(config.gh2_config['pre_infusion'])),
                        1 if config.gh2_config.get('backflush', False) else 0
                    ]
                    if last_gh2_data != new_gh2:
                        send_gh_uart(2, config.gh2_config)
                        last_gh2_data = new_gh2
                # Log the updated configurations
                print("\nUpdated Group Head Configurations:")
                print("--------------------------------")
                print("Group Head 1:")
                print(json.dumps({
                    "temperature": config.gh1_config['temperature'],
                    "pre_infusion": {
                        "enabled": bool(config.gh1_config['pre_infusion'] > 0),
                        "time": config.gh1_config['pre_infusion']
                    },
                    "extraction_time": config.gh1_config['extraction_time'],
                    "volume": config.gh1_config['extraction_volume'],
                    "purge": config.gh1_config['purge'],
                    "backflush": config.gh1_config['backflush']
                }, indent=2))
                print("\nGroup Head 2:")
                print(json.dumps({
                    "temperature": config.gh2_config['temperature'],
                    "pre_infusion": {
                        "enabled": bool(config.gh2_config['pre_infusion'] > 0),
                        "time": config.gh2_config['pre_infusion']
                    },
                    "extraction_time": config.gh2_config['extraction_time'],
                    "volume": config.gh2_config['extraction_volume'],
                    "purge": config.gh2_config['purge'],
                    "backflush": config.gh2_config['backflush']
                }, indent=2))
                print("--------------------------------\n")

            elif self.path == '/setstatusupdate':
                print("\nReceived POST request to /setstatusupdate")
                print("Request data:", json.dumps(params, indent=2))
                target = params.get('target')
                status = params.get('status')
                state_changed = False
                if target == 'main_boiler':
                    new_state = 1 if status else 0
                    if new_state != main_boiler_state:
                        main_boiler_state = new_state
                        state_changed = True
                elif target == 'gh1':
                    new_state = 1 if status else 0
                    if new_state != gh1_button_state:
                        gh1_button_state = new_state
                        state_changed = True
                elif target == 'gh2':
                    new_state = 1 if status else 0
                    if new_state != gh2_button_state:
                        gh2_button_state = new_state
                        state_changed = True
                print(f"Status update: {target} set to {status}")
                # Send main UART string if any button state changed
                if state_changed:
                    new_main = [main_boiler_state, gh1_button_state, gh2_button_state, int(round(config.pressureConfig['pressure'])), int(round(config.mainAmpereConfig['temperature']))]
                    if last_main_data != new_main:
                        send_main_uart()
                        last_main_data = new_main
                self.send_response(200)
                self.send_header('Content-type', 'application/json')
                self.send_header('Access-Control-Allow-Origin', '*')
                self.end_headers()
                self.wfile.write(json.dumps({'success': True}).encode())

            # Send success response
            self.send_response(200)
            self.send_header('Content-type', 'text/html')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.send_header('Access-Control-Allow-Methods', 'POST, GET, OPTIONS')
            self.send_header('Access-Control-Allow-Headers', 'Content-Type')
            self.end_headers()
            self.wfile.write(b'POST request received!')
        except Exception as e:
            print(f"Error processing POST request: {e}")
            self.send_response(500)
            self.send_header('Content-type', 'text/html')
            self.send_header('Access-Control-Allow-Origin', '*')
            self.end_headers()
            self.wfile.write(b'Internal Server Error')

def run_server(port=8000):
    server_address = ('', port)
    httpd = HTTPServer(server_address, RequestHandler)
    print(f'Starting server on port {port}...')
    try:
        httpd.serve_forever()
    except KeyboardInterrupt:
        print("Shutting down server...")
        httpd.server_close()

if __name__ == '__main__':
    run_server() 
