# big_challenges
import imaplib
import smtplib
import pymorphy2
import os
import re
import email
from email.header import decode_header
from email.header import Header
from email.mime.multipart import MIMEMultipart
from email.mime.text import MIMEText
from email.mime.image import MIMEImage
from email.utils import formataddr

# учетные данные
username = "testakk2840@gmail.com"
password = "666777888qwertY"

directory = r'C:\Users\1\Desktop\address'  # расположение папки с отделами


# возврат слова к начальной форме и заполнение списка ими
def norm(x):
    morph = pymorphy2.MorphAnalyzer()
    p = morph.parse(x)[0]
    return p.normal_form


# чистый текст для создания папки
def clean(text):
    return "".join(c if c.isalnum() else "_" for c in text)


# читать только видимые файлы
def onlyVisible(files):
    return list(filter(lambda name: not name.startswith('.'), files))


class EmployeeKeywords:
    def __init__(self, BodyList, KeywordsList):
        self.recipients = self.__getRecipientsByKeywords(BodyList, KeywordsList)
        self.RecipientsEmailAndName = self.__getRecipientsEmailAndName(BodyList, KeywordsList)

    # проверка на ключевые слова
    def __getRecipientsByKeywords(self, BodyList, KeywordsList):
        recipients = set()
        for man in KeywordsList:
            keywords = man[-1]
            if any(words in keywords for words in BodyList):
                recipients.add(man[0])
        return recipients

    # проверка на нахождение в письме имен и почт сотрудников
    def __getRecipientsEmailAndName(self, BodyList, KeywordsList):
        recipients = set()
        for words in BodyList:
            for ListInList in range(len(KeywordsList)):
                for WordInList in range(1, 3):
                    if any(words in KeywordsList[ListInList][WordInList] for check in BodyList):
                        recipients.add(KeywordsList[ListInList][0])
        return recipients

# получение названия файла, имени, email с @.com, email без @.com
class Department:
    def __init__(self, pathToEmployeeFolder):
        self.employees = self.__readEmployees(pathToEmployeeFolder)

    def __readEmployees(self, path):  # Читаем структуру папки, получаем список имен и адресов людей, конструктору класса скармливаем все, включая сам файл
        employees = []
        for department in os.listdir(path):
            for person in os.listdir(os.path.join(path, department)):
                nottxtemplName = os.path.splitext(person)[0]
                (name, email) = nottxtemplName.split("; ")# Декомпозируешь название на имя и емейл, понадобится для создания справочника
                notdogemail = email.split('@')[0]
                employees.extend([os.path.join(path, department, person), name, email, notdogemail])
        return employees


# получение ключевых слов
class Employee:
    def __init__(self, pathToKeywordsFile, name, email, NotDogEmail):
        self.name = name
        self.email = email
        self.NotDogEmail = NotDogEmail
        self.keywords = self.__readKeywords(pathToKeywordsFile)

    def __readKeywords(self, path):# Просто читаем файл, разбивая его на отдельные слова
        with open(path, 'r') as fp:
            return fp.read().split('\n')


# получение текста
def getEmailKeywords(text):
    text = re.findall(r'\w+', text)
    normtext = []
    for a in text:
        normtext.append(norm(a))
    return normtext


File = Department(directory) # получение общего списка с ключевыми словами
File = File.employees
KeywordsStr = [] # почта, ключевые слова и т.д. для определенного человека
KeywordsList = [] # список со всеми почтами, ключевыми файлами и т.д.
FileDirectory = '' # расположение файла с ключевыми словами
name = '' # имя человека
mail = '' # почта человека
score = 0 # нужно для просчета позиции слова в списке
for i in File:
    score += 1
    if i.startswith(directory):
        FileDirectory = i
        continue
    if score % 2 == 0 and score % 4 != 0:
        name = i
        continue
    if score % 3 == 0 and score % 2 != 0:
        mail = i
        continue
    if score == 4:
        j = Employee(FileDirectory, name, mail, i) # в данном случае i - почта без @
        KeywordsStr = [j.email, j.NotDogEmail, norm(j.name), list(map(lambda word: norm(word), j.keywords))]
        KeywordsList.append(KeywordsStr)
        score = 0


imap = imaplib.IMAP4_SSL("imap.gmail.com")  # создание класса IMAP4 с SSL
imap.login(username, password)  # аутентификация
# создать сервер
server = smtplib.SMTP('smtp.gmail.com: 587')
server.starttls()
server.login(username, password)

# выбор ящика писем
status, message = imap.select('INBOX')
unreadcount = imap.status('INBOX', '(UNSEEN)')  #проверяем наличие непрочитанных сообщений
unreadcount = re.findall(r'\d+', str(unreadcount))  # ищем количество непрочитанных сообщений
unreadcount = int(unreadcount[0])
message = re.findall(r'\d+', str(message))  # ищем количество всех сообщений
message = int(message[0])

BodyList = set()  # сюда заносятся тело и тема в начальной форме
for i in range(message, message-unreadcount, -1):
    res, msg = imap.fetch(str(i), "(RFC822)")
    for response in msg:
        if isinstance(response, tuple):
            # получение текста
            msg = email.message_from_bytes(response[1])
            # расшифровка текста
            subject, encoding = decode_header(msg.get("Subject"))[0]
            if isinstance(subject, bytes):
                # перевод в текст
                subject = subject.decode(encoding)
                for d in getEmailKeywords(subject):
                    BodyList.add(d)
                # расшифровка отправителя
            From, encoding = decode_header(msg.get("From"))[0]
            if isinstance(From, bytes):
                From = From.decode(encoding)
                # перебор частей письма
            for part in msg.walk():
                # получение типа содержимого письма
                content_type = part.get_content_type()
                content_disposition = str(part.get("Content-Disposition"))
                if content_type == 'text/plain':
                    body = part.get_payload(decode=True).decode()
                    for d in getEmailKeywords(body):
                        BodyList.add(d)
                    # удаление знаков препинани€
                if "attachment" in content_disposition:
                    # скачать вложение
                    filename = part.get_filename()
                    if filename:
                        folder_name = clean(subject)
                        if not os.path.isdir(folder_name):
                            # сделать папку для этого письма, названную как тема
                            os.mkdir(folder_name)
                        filepath = os.path.join(folder_name, filename)

    # проверка на ключевые слова
    RecipientsList = EmployeeKeywords(BodyList, KeywordsList)
    RecipientsList = RecipientsList.RecipientsEmailAndName # сюда заносится адрес человека с найденными почтой/именем
    if RecipientsList == set():
        RecipientsList = EmployeeKeywords(BodyList, KeywordsList)
        RecipientsList = RecipientsList.recipients # сюда заносится адрес человека с найденными ключевыми словами для него

    if RecipientsList == set():
        continue

    # создание объекта сообщения
    msg = MIMEMultipart()
    # установка параметра сообщения
    msg['From'] = formataddr((str(Header(From, 'utf-8')), 'from@mywebsite.com'))
    msg['To'] = ','.join(RecipientsList)
    msg['Subject'] = subject
    # работа со вложением
    try:
        with open(filepath, 'rb') as fp:  # может стоять encoding='utf-8'
            file = fp.read()
            msg.attach(MIMEImage(file, name=filepath))
    except:
        pass
    msg.attach(MIMEText(body))  # добавить текст
    server.sendmail(msg['From'], msg['To'].split(','), msg.as_string())  # отправить сообщение через сервер
    BodyList.clear()
    RecipientsList.clear()
server.quit()
imap.close()
imap.logout()
