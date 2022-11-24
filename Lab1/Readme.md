# Запуск ЛР
Запустить докер-контейнер следующей командой:
```
docker-compose up --build -d
```
Затем подключиться к виртуальной среде с помощью следующей команды (пароль mapr):
```
ssh root@localhost -p 2222
```
Перейти в директорю с ЛР:
```
cd /home/mapr/lab_1
```

Подготовить среду к работе с помощью следующих команд:
```
echo 'export PATH=$PATH:/opt/mapr/spark/spark-3.2.0/bin' > /root/.bash_profile
source /root/.bash_profile
```
```
apt-get update && apt-get install -y python3-distutils python3-apt
wget https://bootstrap.pypa.io/pip/3.6/get-pip.py
python3 get-pip.py
```
```
pip install jupyter
pip install pyspark
```
Затем запустить jupyter notebook с помощью следующей команды:
```
jupyter-notebook --ip=0.0.0.0 --port=50001 --allow-root --no-browser
```
Выполненные задания находятся внутри ноутбука
