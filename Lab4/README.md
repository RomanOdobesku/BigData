# Lab 4 - Zookeeper 
## Установка и запуск
### В данной ЛР используется kazoo - это как Zookeper (даже докер образ тот же), но для Python
### Как запустить?
1. Запускаем docker-compose: ``` docker-compose up --build -d ```
2. Внутри докер-контейнера запускаем следующие команды для установки всего необходимого:
```
apt-get update && apt-get install -y python3-distutils python3-apt  
wget https://bootstrap.pypa.io/pip/3.6/get-pip.py  
python3 get-pip.py  
pip install jupyter  
pip install kazoo  
```
3. Запустим jupyter ноутбук: ```jupyter-notebook --ip=0.0.0.0 --port=50001 --allow-root --no-browser ``` и переходим по появившейся ссылке. Далее работа ведется через Jupyter notebook

## Task 1. Dining philosophers
### Задание
Решите проблему обедающих философов (каждый философ - отдельный процесс в системе)
### Решение
1. Класс Philosopher эмулирует поведение каждого философа, он содержит в себе информацию о соседних вилках, свой id и id партнёров.
```python
class Philosopher(Process):
	def __init__(self, task_name: str, id: int, fork_path: str, eat_seconds: int = 15, max_id: int = 5):
		super().__init__()
		self.root = task_name
		self.fork = fork_path
		self.id = id
		self.left_fork_id = id
		self.right_fork_id = id + 1 if id + 1 < max_id else 0
		self.eat_seconds = eat_seconds
		self.partner_id_left = id - 1 if id - 1 >= 0 else max_id-1
		self.partner_id_right = id + 1 if id + 1 < max_id else 0
```
2. Одновременно только один философ может брать вилки, это осуществляется с помощью блокировки стола (наиболее общий ресурс)
```python
table_lock = zk.Lock(f'{self.root}/table', self.id)
left_fork = zk.Lock(f'{self.root}/{self.fork}/{self.left_fork_id}', self.id)
right_fork = zk.Lock(f'{self.root}/{self.fork}/{self.right_fork_id}', self.id)
```
3. Будем эмулировать работу программы в течение eat_seconds для каждого философа
```python
start = time()
while time() - start < self.eat_seconds:
```
4."Блокируем вилки если они свободны и соседи поели не меньше чем он сам
```python
with table_lock:
	if len(left_fork.contenders()) == 0 and len(right_fork.contenders()) == 0 \
		and counters[self.partner_id_right] >= counters[self.id] \
		and counters[self.partner_id_left] >= counters[self.id]:
			left_fork.acquire()
			right_fork.acquire()
```
5. Если философ взял вилки, то он поест, сообщит об этом, увеличит свой счетчик пищи и освободит вилки, иначе будет думать
```python
if left_fork.is_acquired and right_fork.is_acquired:
	print(f'{self.id} философ: Я придумал как взять вилки! Поем, пока не забыл...')
	counters[self.id] += 1
	sleep(MEAL_TIME_SEC)
	left_fork.release()
	right_fork.release()
else:
	print(f'{self.id} философ: Как бы взять вилки? Надо подумать...')
	sleep(WAITING_TIME_SEC)
```
6. Создаем всё необходимое и запускаем процессы с философами
```python
root = '/task1'
fork_path = 'forks'
seconds_eat = 20
WAIT_EAT_MS = 1.25
WAIT_AFTER_ALL_DONE = 0.4

master_zk = KazooClient()
master_zk.start()

if master_zk.exists(root):
    master_zk.delete(root, recursive=True)

master_zk.create(root)
master_zk.create('/task1/table')
master_zk.create('/task1/forks')
for i in range(0,5):
    master_zk.create(f'/task1/forks/{i}')

counters = Manager().list()
p_list = list()
for i in range(0, 5):
    p = Philosopher(root, i, fork_path, seconds_eat)
    counters.append(0)
    p_list.append(p)
    
for p in p_list: 
    p.start()
```

#### Результат
Философы едят +- одинаковое количество раз </br>
![image](https://user-images.githubusercontent.com/60855603/205640631-bb9cbcfb-9309-4fa1-9f95-a6277b005c90.png)

### Task 2. Two phase сommit
#### Задание
Реализуйте двуфазный коммит протокол для high-available регистра (каждый регистр - отдельный процесс в системе)
#### Код и описание
1. Класс Client содержит id, путь до своего узла и KazooClient
```python
def __init__(self, root: str, id: int, zk):
	super().__init__()
	self.url = f'{root}/{id}'
	self.root = root
	self.id = id
	self.zk = zk
```
2. В методе watch_myself происходит обработка действий, которые приходят от Coordinator.
```python
def watch_myself(data, stat):
    if data == ACTION_DISCONNECT:
        print(f'Клиент {self.id} уведомлён о том, что один из участников отключился')
    else:
        if(stat.version == 1):
            sleep(1)
        if stat.version != 0:
            print(f'Клиент {self.id} принял решение от координатора: {data.decode()}')
```
3. В методе run происходит случайный выбор действия ACTION/ROLLBACK, а также происходит запуск функции watch_myself (через DataWatch), которая следит за изменениями своего узла. По истечению времени ```WAIT_HARD_WORK_SEC``` клиенты завершают свою работу.
```python
def run(self):      
	self.zk.start()
  value = ACTION_COMMIT if random.random() > 0.5 else ACTION_ROLLBACK
  print(f'Клиент {self.id} запросил {value.decode()}')
  self.zk.create(self.url, value, ephemeral=True)
  datawatcher = DataWatch(self.zk, self.url, watch_myself)

  sleep(WAIT_HARD_WORK_SECONDS)
  self.zk.stop()
  print(f'Клиент {self.id} отключился')
  self.zk.close()
```
4. Coordinator следит за работой всех потоков и содержит в себе единственное поле timer, который срабатывает по расписанию.
5. В методе make_decision выбирается действие методом голосования и результат выбора отправялется каждому клиенту.
```python
Coordinator.timer.cancel()
tr_clients = coordinator.get_children('/task_2/transaction')
commit_counter = 0
abort_counter = 0
for client in tr_clients:
    commit_counter += int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_COMMIT)
    abort_counter +=  int(coordinator.get(f'/task_2/transaction/{client}')[0] == ACTION_ROLLBACK)

# Принимает commit только единогласно
final_action = ACTION_COMMIT if commit_counter == number_of_clients else ACTION_ROLLBACK
for client in tr_clients:
    coordinator.set(f'/task_2/transaction/{client}', final_action) # Рассылаем результат
````
6. Метод check_clients срабатывает по таймеру: получает информацию о клиентах и оповещает других, если один из уже подключенных клиентов отсоединился и рассылает им предупреждение об этом.
```python
def check_clients():
	tr_clients = coordinator.get_children('/task_2/transaction')
  for i in range(len(Coordinator.session_logs)):
      if Coordinator.session_logs[i] is True and str(i) not in tr_clients:
          print("Один участник отключился, всем остальным разослано сообщение с оповещением")
          sleep(0.5)
          Coordinator.timer.cancel()
          for client in tr_clients:
              coordinator.set(f'/task_2/transaction/{client}', ACTION_DISCONNECT)
          sleep(0.5)
          for client in tr_clients:
              zk_list[int(client)].stop()
              zk_list[int(client)].close()
              p_list[int(client)].kill()
          Coordinator.timer.cancel()
          sys.exit()
```
7. Метод watch_clients устанавливает значения в session_logs при первом подключение клиента, а далее происходит обработка количества клиентов.
```python
@coordinator.ChildrenWatch('/task_2/transaction')
def watch_clients(clients):
    for client in clients:
        Coordinator.session_logs[int(client)] = True

    if len(clients) == 0:
        if Coordinator.timer is not None:
            Coordinator.timer.cancel()
    else:
        if Coordinator.timer is not None:
            Coordinator.timer.cancel()
        Coordinator.timer = threading.Timer(duration, check_clients) # Проверяем, не отключился ли клиент
        Coordinator.timer.daemon = True
        Coordinator.timer.start()

    if len(clients) < number_of_clients:
        print(f'Подключенные клиенты:{clients}')
    elif len(clients) == number_of_clients:
        make_decision()
```
8. В методе main создается общий процесс и клиенты
```python
Coordinator.session_logs = [False] * number_of_clients # Храним, заходил ли уже клиент
coordinator = KazooClient()
coordinator.start()

if coordinator.exists('/task_2'):
    coordinator.delete('/task_2', recursive=True)

coordinator.create('/task_2')
coordinator.create('/task_2/transaction')

Coordinator.timer = None
root = '/task_2/transaction'
       
for i in range(number_of_clients):
    zk_list.append(KazooClient())
    p = Client(root, i, zk_list[-1])
    p_list.append(p)
    p.start()
    sleep(7)
```
#### Результат
![image](https://user-images.githubusercontent.com/60855603/205663025-ba1c545c-3a26-43e6-8961-e2154db55e98.png)
