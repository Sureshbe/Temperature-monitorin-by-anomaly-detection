# Temperature-monitorin-by-anomaly-detection
import conf, email_conf, json, time, math, statistics
from boltiot import Sms, Email, Bolt



def compute_bounds(history_data,frame_size,factor):
    if len(history_data)<frame_size :
        return None

    if len(history_data)>frame_size :
        del history_data[0:len(history_data)-frame_size]
    Mn=statistics.mean(history_data)
    Variance=0
    for data in history_data :
        Variance += math.pow((data-Mn),2)
    Zn = factor * math.sqrt(Variance / frame_size)
    High_bound = history_data[frame_size-1]+Zn
    Low_Bound = history_data[frame_size-1]-Zn
    return [High_bound,Low_Bound]

minimum_limit = 4  
maximum_limit = 7

mybolt = Bolt(email_conf.API_KEY, email_conf.DEVICE_ID)
mailer = Email(email_conf.MAILGUN_API_KEY, email_conf.SANDBOX_URL, email_conf.SENDER_EMAIL, email_conf.RECIPIENT_EMAIL)
sms = Sms(conf.SSID, conf.AUTH_TOKEN, conf.TO_NUMBER, conf.FROM_NUMBER)

history_data=[]

while True:
    response = mybolt.analogRead('A0')
    data = json.loads(response)
    
    if data['success'] != '1':
        print("There was an error while retriving the data.")
        print("This is the error:"+data['value'])
        time.sleep(10)
        continue

    
    print ("The sensor value is "+(data['value']))
    sensor_value=0
    try:
        sensor_value = int(data['value'])
        
    
    except Exception as e:
        print("There was an error while parsing the response: ",e)
        continue

    bound = compute_bounds(history_data,email_conf.FRAME_SIZE,email_conf.MUL_FACTOR)
    if not bound:
        required_data_count=email_conf.FRAME_SIZE-len(history_data)
        print("Not enough data to compute Z-score. Need ",required_data_count," more data points")
        history_data.append(int(data['value']))
        time.sleep(10)
        continue

    try:
        if sensor_value > maximum_limit or sensor_value < minimum_limit:
            response = mailer.send_email("Alert", "The Current temperature is beyond the threshold ")
        if sensor_value > bound[0] :
            print ("The temperature increased suddenly. Sending Sms.")
            response = sms.send_sms("Someone opened the chamber")
            print("This is the response ",(response))
        elif sensor_value < bound[1]:
            print ("The temperature decreased suddenly. Sending an email.")
            response = mailer.send_email("Someone opened the chamber")
            print("This is the response ",response)
        history_data.append(sensor_value);
    except Exception as e:
        print ("Error",e)
    time.sleep(10)
