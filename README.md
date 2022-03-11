# WebScraping
Задача: Получить и сохранить данные со сторонних ресурсов для дальнейшего использования.

<a name="3"></a>
## [Оглавление:](#3)
1. [Requests через API](#1)
2. [BeautifulSoup через HTML](#2)


Импортируем нужные библиотеки.
```
!pip install -q tqdm
```
```
import requests                                         # Библиотека для формирования запросов
import csv                                              # Для работы с csv
import datetime                                         # Для подсчета времени 
import pandas as pd                                     # Для преобразования данных в удобный для просмотра вид
import tqdm                                             # Импорт tqdm
from google.colab import auth                           # Авторизация
from googleapiclient.discovery import build             # googleapiclient build
import io                                               # Импорт io
from googleapiclient.http import MediaIoBaseDownload    # googleapiclient MediaIoBaseDownload
from googleapiclient.http import MediaFileUpload        # googleapiclient MediaFileUpload
```
[:arrow_up:Оглавление](#3)
<a name="1"></a>
## Requests через API.
```
reviews = []    # Лист для комментариев 
comments = {}   # Словарь комментариев 
```
Функции парсера.
```
def time_counter(start_time, now_time, count, amount):
    print('Прошло: ', now_time - start_time, '. Осталось: ', ((now_time - start_time)/count)*(amount - count), sep='')
```
```
def get_courses(): 
    '''
    Функция для получения списка курсов 
    '''
    i = 1       # Начальный номер страницы парсинга 
    ids = []    # Все id курсов 

    while True: # Бесконечный цикл, который прервется только в том случае, если метапараметр покажет отсутствие следующей страницы 
        print('Получение списка курсов на странице {}'.format(i))
        data = {'is_cataloged': 'true',
                'order': '-activity',
                'page': i} # Передаваемые параметры для запроса.

        r = requests.get('https://stepik.org/api/courses', params=data).json() # Осуществляем запрос, читаем ответ и преобразуем в формат json
        # Формат нового запроса 'https://stepik.org/api/courses?page=1&is_cataloged=true&order=-activity'
        
        for course in r['courses']:     # Все полученные id курсов для данной страницы 
            ids.append(course['id'])    # Добавляем в общий список ids 
            
        i += 1  # увеличиваем i для перехода к следующей странице

        if not r['meta']['has_next']:   # Прерываем бесконечный цикл в случе, если метапараметр показывает отсутствие следующей страницы 
            return ids
```
```
def get_course_reviews(id, start_time, val_id, val_courses): 
    '''
    Функция для получения комментариев по каждому из списка курсов
    ''' 
    i = 1       # Начальный номер страницы парсинга 

    while True: # Бесконечный цикл, который прервется только в том случае, если метапараметр покажет отсутствие следующей страницы 
        time_counter(start_time, datetime.datetime.now(), val_id, val_courses)          # Считаем время выполнения кода для каждой страницы
        print('Получение комментариев для курса {} страницы {}'.format(id, i))
        data = {'course': id, 'page': i}                                                # Передаваемые параметры для запроса.

        r = requests.get('https://stepik.org/api/course-reviews', params=data).json()   # Осуществляем запрос, читаем ответ и преобразуем в формат json
        
        for comment in r['course-reviews']:                # В полученном ответе 
            reviews.append({'course': comment['course'],   # все id курсов
                        'users_id': comment['user'],       # id пользователей 
                        'review': comment['text'],         # тексты комментариев 
                        'date': comment['update_date']})   # и даты написания комментариев добавляем в список комментариев
        i += 1  # увеличиваем i для перехода к следующей странице

        if not r['meta']['has_next']: # Прерываем бесконечный цикл в случе, если метапараметр показывает отсутствие следующей страницы 
            return reviews
```
```
def csv_writer(FILENAME): 
    '''
    Функция для записи данных в scv. FILENAME - имя для нового файла
    ''' 
    with open(FILENAME, "w", newline="" , encoding='utf-16') as file:       # Создаем новый файл scv
        columns = ["course","users_id", "review","date"]                    # Определяем названия столбцов 
        writer = csv.DictWriter(file, delimiter=',', fieldnames=columns)    # Создаем экземпляр DictWriter и передаем ему объект файла, значение разделителя и наш список наименований полей
        writer.writeheader()                                                # Записываем названия столбцов в файл
        writer.writerows(reviews)                                           # Записываем комментарии
```
```
def save_file_to_google_drive(local_filename, dest_filename, mimetype = 'application/octet-stream'):
    '''
    Функция для записи данных в Google Drive
    '''
    auth.authenticate_user()
    drive_service = build('drive', 'v3')

    file_metadata = {
        'name': dest_filename,
        'mimeType': mimetype
    }
    media = MediaFileUpload(local_filename, 
                            mimetype=mimetype,
                            resumable=True)
    created = drive_service.files().create(body=file_metadata,
                                            media_body=media,
                                            fields='id').execute()
    print('File ID: {}'.format(created.get('id')))
    return created.get('id')
```
```
def api_parser():
  name = 'Stepik_Output.csv'            # Имя для нового файла
  start_time = datetime.datetime.now()  # Определяем время старта программы
  print('Время запуска парсера:', start_time)

  ids = get_courses()                   # Получаем id всех курсов 
  val_courses = len(ids)                # Считаем общее колличество найденных курсов
  print('Число найденных курсов: ', val_courses)

  for _id in ids:   # По каждому id 
    comments[_id] = get_course_reviews(_id, start_time, ids.index(_id) + 1, val_courses) # Читаем комментари  

  csv_writer(name)  # Записываем данные в scv
  save_file_to_google_drive(name, name) # Записываем scv файл в Google Drive. Первый параметр - имя файла, второй - имя, под которым мы хотим сохранить файл

  end_time = datetime.datetime.now()    # Определяем время окончания программы
  print(end_time - start_time)          # Считаем сколько времени ушло на выполнение кода 
```
Запускаем парсер.
```
api_parser()
```
Прочтем из scv записанные данные.
```
df = pd.read_csv('Stepik_Output.csv', sep=',', encoding = 'UTF_16')
df # Выводим таблицу
```
[:arrow_up:Оглавление](#3)
<a name="2"></a>
## BeautifulSoup через HTML
Импорт библиотек.
```
import requests                 # Импортируем для формирования запрсов в Интернет 
from bs4 import BeautifulSoup   # Для парсинга 
import pandas as pd             # Для преобразования данных в удобный для просмотра вид 
import datetime                 # Для подсчета времени 
import csv                      # Для работы с csv
```
Функции парсера.
```
ames_list = []                 # Список для всех матчей
```
```
def time_counter(start_time, now_time, count, amount):
  print('Прошло: ', now_time - start_time, '. Осталось: ', ((now_time - start_time)/count)*(amount - count), sep='')
```
```
def get_inform(start_time):
    '''
    Функция для получения иформации о матчах
    ''' 
    pages = 525                         # Число страниц парсинга
    i = 0                               # Номер стартовой страницы

    for i in range(pages):              # Проходим по всем страницам
        print(i , '/', pages, end= ' ') # Печатаем прогресс выполненной работы
        time_counter(start_time, datetime.datetime.now(), i + 1, pages) # Считаем сколько времени осталось 
        data = {'offset': i * 100}      # Передаваемые параметры для запрсоов
        r = requests.get('https://www.hltv.org/results?', params=data)  # Формируем запрос 

        soup = BeautifulSoup(r.text, features="html.parser")            # Парсим полученный html

        games = soup.findAll('div', {'class' : 'result-con'})           # Находим все блоки с матчами
        for game in games:
            if game.findAll('div', {'class': 'team team-won'}) != []:   # Если в игре не ничья 
                # Добавялем словарь с информацией о игре в список
                games_list.append({'Gamer1': game.findAll('div', {'class': 'team team-won'})[0].text,
                              'Score1': game.findAll('span', {'class': 'score-won'})[0].text,
                              'Score2': game.findAll('span', {'class': 'score-lost'})[0].text,
                              'Gamer2': game.findAll('div', {'class': 'team '})[0].text})

            else:
                # Добавялем словарь с информацией о игре в список другим образом, если в игре ничья
                games_list.append({'Gamer1': game.findAll('div', {'class': 'team '})[0].text,
                      'Score1': game.findAll('span', {'class': 'score-tie'})[0].text,
                      'Score2': game.findAll('span', {'class': 'score-tie'})[1].text,
                      'Gamer2': game.findAll('div', {'class': 'team '})[1].text})         
```
```
def csv_writer(FILENAME):
    '''
    Функция для записи данных в scv. FILENAME - имя для нового файла 
    '''
    with open(FILENAME, "w", newline="" , encoding='utf-16') as file:       # Создаем новый файл scv
        columns = ['Gamer1','Score1', 'Score2', 'Gamer2']                   # О пределяем названия столбцов 
        writer = csv.DictWriter(file, delimiter=',', fieldnames=columns)    # Создаем экземпляр DictWriter и передаем ему объект файла, значение разделителя и наш список наименований полей
        writer.writeheader()                    # Записываем в названия столбцов в файл
        writer.writerows(games_list)            # Записываем комментарии
```
```
def csv_writer(FILENAME):
    '''
    Функция для записи данных в scv. FILENAME - имя для нового файла 
    '''
    with open(FILENAME, "w", newline="" , encoding='utf-16') as file:       # Создаем новый файл scv
        columns = ['Gamer1','Score1', 'Score2', 'Gamer2']                   # О пределяем названия столбцов 
        writer = csv.DictWriter(file, delimiter=',', fieldnames=columns)    # Создаем экземпляр DictWriter и передаем ему объект файла, значение разделителя и наш список наименований полей
        writer.writeheader()                    # Записываем в названия столбцов в файл
        writer.writerows(games_list)            # Записываем комментарии
```
```
def html_parser():
  name = "hltv_Output.csv"                      # Определяем имя файлу scv
  start_time = datetime.datetime.now()          # Определяем время запуска парсера 
  print('Время запуска парсера:', start_time)
  get_inform(start_time)                        # Получаем ифнормацию о всех играх 
  
  csv_writer(name)                              # Записываем файл scv
  save_file_to_google_drive(name, name)         # Сохраняем файл в Google Drive
```
Запускаем парсер.
```
html_parser()
```

[:arrow_up:Оглавление](#3)

[Ноутбук](https://colab.research.google.com/drive/1oWy8dT5TYUPZ8ioZW6BizCB8FlfJcuB8?usp=sharing)
