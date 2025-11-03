# Enviro-Monitor

Creating a complete Python program for a web application like Enviro-Monitor, which involves real-time environmental monitoring and alerting using IoT sensors and data visualization, is quite an extensive task. Below is a simplified version of what such a project might look like. This program involves a Flask web server, which will serve web pages, an MQTT client for receiving sensor data, and Matplotlib for data visualization. Bear in mind that this is a skeleton version; a production-ready application would require more robust features and optimizations.

You'll need to install some Python packages to run this code: Flask for the web app, Paho-MQTT for the MQTT client, and Matplotlib for plotting data. You can install them using pip:

```bash
pip install flask paho-mqtt matplotlib
```

Here's a complete, simplified Python program:

```python
from flask import Flask, render_template, jsonify
import paho.mqtt.client as mqtt
import threading
import time
from datetime import datetime
import matplotlib.pyplot as plt
import os

# Define the Flask application
app = Flask(__name__)

# Global variable to store environmental data
sensor_data = []

# MQTT Broker details
BROKER_HOST = "iot.eclipse.org"
BROKER_PORT = 1883
TOPIC = "enviro/monitor"

# Setting path for Matplotlib to avoid X server issues
os.environ['MPLCONFIGDIR'] = "/tmp"

# Function to plot data
def plot_data():
    timestamps = [data['timestamp'] for data in sensor_data]
    values = [data['value'] for data in sensor_data]
    
    plt.figure(figsize=(10, 5))
    plt.plot(timestamps, values, marker='o')
    plt.title('Environmental Data Over Time')
    plt.xlabel('Time')
    plt.ylabel('Sensor Value')
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.savefig('static/plot.png')
    plt.close()

# MQTT Client Setup and Callbacks
def on_connect(client, userdata, flags, rc):
    if rc == 0:
        print("Connected to MQTT Broker")
        client.subscribe(TOPIC)
    else:
        print(f"Failed to connect, return code {rc}")

def on_message(client, userdata, msg):
    try:
        print(f"Message received: {msg.payload.decode()}")
        data_value = float(msg.payload.decode())
        sensor_data.append({'timestamp': datetime.now().strftime('%H:%M:%S'), 'value': data_value})
        
        if len(sensor_data) > 100:  # Save up to the last 100 readings
            sensor_data.pop(0)
        
        plot_data()
    except ValueError as e:
        print(f"Error decoding message: {e}")

def start_mqtt_client():
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message
    
    try:
        client.connect(BROKER_HOST, BROKER_PORT, 60)
        client.loop_start()
    except Exception as e:
        print(f"Error connecting to MQTT broker: {e}")

# Flask Routes
@app.route('/')
def index():
    # Serve the main page with data visualization
    return render_template('index.html')

@app.route('/data')
def get_data():
    # Serve the latest sensor data
    try:
        return jsonify(sensor_data)
    except Exception as e:
        return jsonify({"error": f"Failed to retrieve data: {e}"})

# HTML Template (would be in 'templates/index.html')
index_html = """
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Enviro-Monitor</title>
</head>
<body>
    <h1>Enviro-Monitor Dashboard</h1>
    <div>
        <img src="{{ url_for('static', filename='plot.png') }}" alt="Environmental Data Plot">
    </div>
</body>
</html>
"""

# Add the HTML template to Flask's template folder
if not os.path.exists('templates'):
    os.mkdir('templates')
with open('templates/index.html', 'w') as f:
    f.write(index_html)

# Entry point for starting the app and MQTT client
if __name__ == '__main__':
    mqtt_thread = threading.Thread(target=start_mqtt_client)
    mqtt_thread.start()

    try:
        app.run(host='0.0.0.0', port=5000, debug=True)
    except Exception as e:
        print(f"Error starting Flask app: {e}")
```

### Explanation:

1. **Flask Setup:** The `Flask` module is used to serve the web page that displays the environmental data plot.

2. **MQTT Client Configuration:** `paho.mqtt.client` is configured to connect to a public broker and subscribe to a topic. Replace `BROKER_HOST`, `BROKER_PORT`, and `TOPIC` with specifics from your IoT setup.

3. **Data Processing and Plotting:** Incoming data is stored and plotted using Matplotlib. The resulting plot is saved as `static/plot.png`. The data is visualized on a web page served by Flask, which uses HTML templates.

4. **Error Handling:** The program includes simple error handling for exceptions that might occur during MQTT message processing, data parsing, and web server startup.

5. **Concurrency:** A separate thread runs the MQTT client to allow continuous data collection while the web server handles HTTP requests.

Please note, deploying such an application for real-world use involves setting up authentication, secure communications (HTTPS, secure MQTT connections), and much more robust data handling and storage mechanisms (like using a real database).