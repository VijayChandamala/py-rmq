# py-rmq
---------
RabbitMQ provides a REST API for getting info about vhosts,queues and messages in the queue
https://pulse.mozilla.org/api/

using which we'll be getting whether a queue has messages waiting and will spawn a listener (docker container)
to consume the messages.
Containers running for more than 100 seconds without any messages in the related queue will be removed.

## commands for getting queues and their message count
---------------------------------------------------
getting queues:

```
response = requests.get('http://rmq_username:rmq_passwd@localhost:15672/api/queues/%s' % vhost)
queues = [format(queue['name']) for queue in response.json()]
```
getting message count based on queue:

```
resp = requests.get('http://rmq_username:rmq_passwd@localhost:15672/api/queues/%s/%s' %(vhost,queue))
data = resp.json()
message_count = format(data['messages'])
```

## Full code
-------------

```
import docker
import requests
import json
import datetime
import subprocess
datetimeFormat = '%Y-%m-%d %H:%M:%S.%f'

client = docker.from_env()
client.containers.list()


############ Mapping queues to rake task #################

mapping={
"queue_name":"container_listener_command"
}

dump = json.dumps(mapping)
qmap = json.loads(dump)

############# Initializing container_uptime and container_status ( 0 = not running, 1 = running ) ##############

container_uptime=0
container_status=1

############ Method to run/remove container based on message_count and container_status #################

def invoke_container_manager(message_count, container_name, run_command, container_uptime,container_status):
    if message_count!= 0:
        if container_status == 0:
            print("%s messages waiting in queue,Running listener: %s" %(message_count,run_command))
            with open("txt.log", "a") as output:
                subprocess.call("%s" % run_command, shell=True, stdout=output, stderr=output)
        else:
            print("%s messages are being consumed by listener: %s" %(message_count,run_command))
    elif message_count == 0:
        if container_status != 0:
            if container_uptime > 100:
                print("%s messages in queue, dropping container %s" %(message_count,container_name))
                with open("txt.log", "a") as output:
                    subprocess.call("docker rm -f %s" % container_name, shell=True, stdout=output, stderr=output)
            else:
                print("waiting for messages, will wait %s more seconds"% (100-container_uptime))
        else:
            print("Great! no messages, no container")


vhost="vhost" ######### defining vhost #########

############ Get queues from RMQ REST API #############

response = requests.get('http://rmq_username:rmq_passwd@localhost:15672/api/queues/%s' % vhost)
queues = [format(queue['name']) for queue in response.json()]

########### Get message count in all queues and check if the related container status/uptime ################

for queue in queues:
    resp = requests.get('http://rmq_username:rmq_passwd@localhost:15672/api/queues/%s/%s' %(vhost,queue))
    data = resp.json()
    message_count = format(data['messages'])
    container_name = vhost.upper()+'-listener-'+qmap[queue]
    run_command=("docker run -id --name %s image_name %s"% (container_name,qmap[queue]))
    try:
        container = client.containers.get("%s" % container_name)
    except docker.errors.NotFound:
        container_status=0;
    if container_status!=0:
        if container.attrs['State']['Status']=='running':
            tsi=container.attrs['State']['StartedAt']
            tsj= tsi.replace("T"," ")
            ts1=tsj.replace("Z","")
            ts=float(datetime.datetime.now().strftime('%s.%f'))
            ts2=str(datetime.datetime.fromtimestamp(ts))
            ts1=ts1[:-6]
            ts1 = datetime.datetime.strptime(ts1, datetimeFormat) 
            ts2 = datetime.datetime.strptime(ts2, datetimeFormat)
            diff = ts2 - ts1
            container_uptime=diff.seconds
            container_status=1
        else:
            container_status=0
    print("message_count=%s,container_name:%s,container_uptime=%s,container_status=%s" % (message_count,container_name,container_uptime,container_status))
    invoke_container_manager(message_count, container_name, run_command, container_uptime, container_status)
```

  Refer Docker sdk for python
    
