# FIWARE Infrastructure — Remote Patient Monitoring (thesis)

Requires Docker Desktop, Node.js, and the FIWARE Device Simulator:
    git clone https://github.com/telefonicaid/fiware-device-simulator.git
    cd fiware-device-simulator && npm install

Start the stack:      docker compose up -d
Create subscription:  curl -X POST http://localhost:1026/v2/subscriptions
                      -H "Content-Type: application/json"
                      -H "fiware-service: hospital" -H "fiware-servicepath: /vitals"
                      -d "@subscription-cygnus.json"
Run the simulator:    nodemon --watch my-simulation.json
                      --exec "node bin/fiwareDeviceSimulatorCLI -c my-simulation.json"