#GENERIC IMPORTS
import os
import time
import board
import busio
import digitalio
import pyrebase
import random

#SENSOR IMPORTS
import adafruit_lps2x
import adafruit_bmp280
import adafruit_sgp30
import adafruit_ahtx0

ADMIN_EMAIL = "n01286513@humbermail.ca"
ADMIN_PASS = "HardWare123"

def firebase_init():
    #FIREBASE CONFIG
    config = {
        "apiKey" : "AIzaSyD5tCiAtyHr0BFozu4ckYbJEIZrFMeJaaQ ",
        "authDomain": "ceng317weatherstationproject.firebaseapp.com",
        "databaseURL" : "https://ceng317weatherstationproject"\
            "-default-rtdb.firebaseio.com/",
        "storageBucket" : "ceng317weatherstationproject.appspot.com"
    }

    #DECLARE FIREBASE OBJECT
    firebase = pyrebase.initialize_app(config)
    return firebase
    
def db_init(firebase):
    database = firebase.database()
    return database

def admin_init(firebase):
    auth = firebase.auth()
    admin = auth.sign_in_with_email_and_password(ADMIN_EMAIL, ADMIN_PASS)
    print("Logged in administrator account")
    return admin

def deviceID_init(db, admin):
    # Check if directory exists, create if not
    if not os.path.isdir("/home/pi/HWS"):
        os.mkdir("/home/pi/HWS")
    os.chdir("/home/pi/HWS")
    
    # Check if device_ID file exists, and if it contains anything
    if not os.path.isfile("device_ID") or os.stat("device_ID").st_size == 0:
        print("No ID found")
        
        first_ID = True
        
        # Get a list of all current IDs on the database
        ID_db = db.child("Weather Station Reading").get(admin['idToken'])
        if ID_db.each() is not None:
            ID_list = list(map(lambda n: n.key(), ID_db.each()))
            first_ID = False
        
        fp = open("device_ID", 'w')
        print("Generating new ID")
        
        # Generate an ID. IDs are 8 hex digits long
        device_ID = hex(random.randint(0, 4294967295))[2:]\
                    .rjust(8, '0').upper()
        
        # Check if new ID matches any IDs already in the database
        # Continue generating until a unique ID is found
        if not first_ID:
            while device_ID in ID_list:
                print("Duplicate ID: " + device_ID)
                print("Generating new ID")
                device_ID = hex(random.randint(0, 4294967295))[2:]\
                            .rjust(8, '0').upper()
        
        print("New ID: " + device_ID)
        fp.write(device_ID)
        fp.close()
        
    # Read and return the device_ID
    fp = open("device_ID", 'r')
    return fp.read()

def sensor_init():
    # Create a dict of the sensors
    # First element of the array is whether the sensor is unresponsive
    # Second element has to do with the LED error codes
    # Third element will be the sensor object, appended to the end
    sensor_array = {
        "lps": [False, 2],
        "bmp": [False, 3],
        "sgp": [False, 4],
        "aht": [False, 5]
    }
    
    # Setup I2C bus
    i2c = busio.I2C(board.SCL, board.SDA)
    
    # Try creating sensor objects. Flip the boolean if it fails
    try:
        lps22 = adafruit_lps2x.LPS22(i2c)
        sensor_array["lps"].append(lps22)
    except:
        print("LPS22 missing or damaged")
        sensor_array["lps"][0] = True
    try:
        bmp280 = adafruit_bmp280.Adafruit_BMP280_I2C(i2c)
        sensor_array["bmp"].append(bmp280)
    except:
        print("BMP280 missing or damaged")
        sensor_array["bmp"][0] = True
    try:
        sgp30 = adafruit_sgp30.Adafruit_SGP30(i2c)
        sensor_array["sgp"].append(sgp30)
    except:
        print("SGP30 missing or damaged")
        sensor_array["sgp"][0] = True
    try:
        aht20 = adafruit_ahtx0.AHTx0(i2c)
        sensor_array["aht"].append(aht20)
    except:
        print("AHT20 missing or damaged")
        sensor_array["aht"][0] = True
    
    # Print the hex addresses of any I2C devices
    # This is mostly for debugging
    print("I2C addresses found:",
          [hex(device_address) for device_address in i2c.scan()])
    return sensor_array

def led_init():
    # Initialize the status LED
    print("Setting up GPIO17 for the LED")
    led = digitalio.DigitalInOut(board.D17)
    led.direction = digitalio.Direction.OUTPUT
    
    # Turn it off in case it was left on
    led.value = False
    return led

def get_readings(sensor_array):
    # Create a dict to hold the readings
    # Default values are there in case a sensor is missing
    enviro_readings = {
        "pressure": 0,
        "temperature": 999,
        "CO2": 0,
        "humidity": 0
    }
    
    # Get the readings from the sensors
    # Leave the default value if they're missing
    if not sensor_array["lps"][0]:
        enviro_readings["pressure"] = float("{:.3f}".format(
            sensor_array["lps"][2].pressure))
    if not sensor_array["bmp"][0]:
        enviro_readings["temperature"] = float("{:.2f}".format(
            sensor_array["bmp"][2].temperature))
    if not sensor_array["sgp"][0]:
        enviro_readings["CO2"] = float("{:.2f}".format(
            sensor_array["sgp"][2].eCO2))
    if not sensor_array["aht"][0]:
        enviro_readings["humidity"] = float("{:.2f}".format(
            sensor_array["aht"][2].relative_humidity))
    
    return enviro_readings

def upload_readings(db, enviro_readings, device_ID, admin):
    # Upload current readings under the device ID
    # These will be overwritten each time
    db.child("Weather Station Reading").child(device_ID)\
      .child("current_readings").child("current_pressure")\
      .set(enviro_readings["pressure"], admin['idToken'])
    db.child("Weather Station Reading").child(device_ID)\
      .child("current_readings").child("current_temperature")\
      .set(enviro_readings["temperature"], admin['idToken'])
    db.child("Weather Station Reading").child(device_ID)\
      .child("current_readings").child("current_CO2")\
      .set(enviro_readings["CO2"], admin['idToken'])
    db.child("Weather Station Reading").child(device_ID)\
      .child("current_readings").child("current_humidity")\
      .set(enviro_readings["humidity"], admin['idToken'])
    
    # Generate a string from the current time
    timestamp = time.strftime("%d%b%Y_%H:%M:%S", time.localtime())
    
    data = {"timestamp": timestamp,
            "pressure_reading": enviro_readings["pressure"],
            "temperature_reading": enviro_readings["temperature"],
            "CO2_reading": enviro_readings["CO2"],
            "humidity_reading": enviro_readings["humidity"]}
    
    # Upload readings under the device ID, using the timestamp as a key
    db.child("Weather Station Reading").child(device_ID)\
      .child("historical_readings").push(data, admin['idToken'])

def print_readings(enviro_readings):
    # Print the readings to the console, mostly for debugging purposes
    print("Atmospheric Pressure: {} hPa". format(enviro_readings["pressure"]))
    print("Temperature: {} C". format(enviro_readings["temperature"]))
    print("CO2 (Air Quality) Value: {} ppm". format(enviro_readings["CO2"]))
    print("Humidity: {} %rH". format(enviro_readings["humidity"]))

def blink_led(status_array, led):
    inverted = False
    
    # Invert the LED pattern if all sensors are missing
    if all(status_array[sensor_name][0] for sensor_name in status_array):
        num_blinks = 1
        inverted = True
    else:
        # Change the number of blinks if a sensor is missing
        # The number associated with each sensor is defined in sensor_array
        # In normal operation blink once
        for sensor_name in status_array:
            if status_array[sensor_name][0]:
                num_blinks = status_array[sensor_name][1]
                break
            else:
                num_blinks = 1
    
    # Each blink cycle should add up to 3 seconds
    off_time = 3.0
    i = 0
    while i < num_blinks:
        led.value = not inverted
        time.sleep(0.2)
        led.value = inverted
        time.sleep(0.2)
        off_time -= 0.4  # Reduce the remaining time to keep it consistent
        i += 1
    time.sleep(off_time)

def firebase_error(led):
    # I couldn't pin down a specific exception to catch, so this
    # runs if anything goes wrong in the functions dealing with firebase
    print("Error connecting to firebase")
    while True:
        led.value = True
        time.sleep(0.2)
        led.value = False
        time.sleep(0.2)
        led.value = True
        time.sleep(0.5)
        led.value = False
        time.sleep(0.6)

# Initializing everything
led = led_init()
try:
    firebase = firebase_init()
    db = db_init(firebase)
    admin = admin_init(firebase)
except:
    firebase_error(led)
dID = deviceID_init(db, admin)
sensor_array = sensor_init()

# Main loop
try:
    print("Initiating Readings...")
    while True:
        enviro_readings = get_readings(sensor_array)
        try:
            upload_readings(db, enviro_readings, dID, admin)
        except:
            firebase_error(led)
        print_readings(enviro_readings)

        blink_led(sensor_array, led)

except KeyboardInterrupt:
    print("\nProgram is shut off by CTRL+C command."
          "Cleaning all GPIO connections.\n")
    led.value = False
    
except:
    # For exceptions without specific handling
    print("\nUnknown Error")
    while True:
        led.value = True
        time.sleep(0.5)
        led.value = False
        time.sleep(0.5)
