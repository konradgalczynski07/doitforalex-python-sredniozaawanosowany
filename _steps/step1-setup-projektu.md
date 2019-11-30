---
layout: step
number: 1
title: Setup projektu
permalink: step1/
---

Teraz przejdziemy do postawienia naszego środowiska developerskiego a następnie omówimy początkową strukturę projektu.

Powinieś mieć już zainstalowane i gotowe do użytku poniższe narzędzia: 

- edytor kodu VSCode lub Pycharm (możesz używać dowolnego edytora jednak nie gwarantujemy ze mentorzy będą wstanie pomóc w przypadku innych edytorów)
- przeglądarkę Chrome z wtyczka [ModHeader](https://chrome.google.com/webstore/detail/modheader/idgpnmonknjnojddfkpgkljpfnnfcklj)
- zainstalowanego Docker'a i Docker Compose - sprawdz to za pomoca komendy `docker -v` oraz `docker-compose -v`

Jeśli wszystko się zgadza możemy przejść dalej. 

Na nasze szczęscie cała infrastruktura oraz ustawienia projektu zostały już zrobione a więc bedziemy mogli bardzo szybko zabrać się za kodowanie nowych ficzerów. Przejdźmy zatem pod poniższy link i zastosujmy się do instrukcji podanej w README.  

```
https://github.com/konradgalczynski07/doitforalex-instagram-backend-end
```

## Pliki startowe

```
doitforalex-instagram-backend
└───instagram-backend
│   │
│   └───instagram
│   │   │   __init__.py
│   │   │   settings.py
│   │   │   urls.py
│   │   │   wsgi.py
│   │   
│   └───users
│   │   │   urls.py
│   │   │   views.py
│   │   │   models.py
│   │   │   ...
│   │   avatar.png
│   │   manage.py  
│ .dockerignore
│ .env  
│ .env_example  
│ .gitignore  
│ docker-compose.yml
│ Dockerfile  
│ Makefile  
│ requirements.txt  
│ wait_for_postgres.py
```

Powyżej znajduje się list plików, więkoszość omówiomy krótko skupiając sie na najważniejszych.

`wait_for_postgres.py` - skrypt odpowiedzialny za sprawdzenie czy baza postgresowa jest już czynna i dostępna dla naszej aplikacji.

`requirements.txt` - plik z wymaganymi paczkami do zainstalowania, inaczej zależnościami (ang. dependencies)

`Makefile` - jak wiecie programiscie lubią być leniwi, plik Makefile sprawia ze długie komendy do wpisania w konsoli można bardzo skrócic, definujemy tam alias który możemy potem użyć do odpalenia danej komendy np. jeśli chcemy zaimportować zmienne środowiskowe z pliku .env możemy użyc krótkiego `make environ` zamiast trudnego do zapamietania `set -o allexport; source .env; set +o allexport`. Polecamy się z nim dobrze zapoznać oraz wracać w chwili potrzeby wpisania jakieś komendy.

`Dockerfile` - tutaj definujemy bazowy docker image z którego chcemy korzystać. W naszym przypadku bedzie to Python3.8.0

`docker-compose.yml` - docker compose pozwala uruchamiać inne image w naszym bazowym dockerze, są tam zdefiniowane 3 kontenery: postgres, frontend i backend. To dzięki niemu będziemy mieli połacznie z baza a frontend dostępny na localhost:3000  

`.gitignore` - plik mowiący git'owi które pliki wykluczyć z systemu kontorli wersji

`.env_exmaple` - przykładowy plik dający nam informacje jakie zmienne środowiskowe powinny znaleźć się w lokalnym pliku .env

`.env` - plik którego nie chcemy wrzucać do naszego repozytorium, to tutaj ustawiany zmienne takie jak djangowy SECRET_KEY

`instagram-backend` - w tym folderze znajduje się projekt Django, poza folderem z ustawieniami znajduje się tam również pierwsza aplikacja users, niestety jest pusta ale na to przyjdzie jeszcze czas

## Settingsy

W plikach Django jedyne zmiany zostały wykonane w pliku `settings.py` omówmy je więc. 

```
SECRET_KEY = os.environ.get('SECRET_KEY')

DEBUG = os.environ.get('DEBUG', False)

ALLOWED_HOSTS = os.environ.get('ALLOWED_HOSTS', '').split(',')

CORS_ORIGIN_WHITELIST = os.environ.get('CORS_ORIGIN_WHITELIST', '').split(',')
```

Powyżej mamy ustawienia których nie chcemy "hardcodowac". Ze względów bezpieczeństwa chcemy mieć możliwość wczytać je ze zmiennych środowiskowych tak żeby nie były one dostępne w repozytorium. Tak również stało się z danymi bazy danych.

Poza tymi zmianami odpowiednie aplikacje zostały dodane do `INSTALLED_APPS` i `MIDDLEWARE`. Paczka 'corsheaders' umożliwi aplikacji Reactowej dostępnej na porcie 3000 na łaczenie się z backendem Django na porcie 8000.

Ponadto zostały ustawione opcje dla DRF'a:

```
# Django Rest Framework

REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.LimitOffsetPagination',
    'PAGE_SIZE': 9,
    'DEFAULT_PERMISSION_CLASSES': ('rest_framework.permissions.IsAuthenticated',),
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
        'rest_framework.authentication.SessionAuthentication',
        'rest_framework.authentication.BasicAuthentication',
    ),
}
```

`DEFAULT_PAGINATION_CLASS` - określa z jakiej paginacji bedzie domyslnie korzystał każdy z widoków (paginacja - porcjowanie elementów zwracanych przez endpoint tak żeby nie wysyłać na raz ogromnej ilosci danych). 

`PAGE_SIZE` - to ilość tych elementów zwracanych przez paginacje przy jednym zapytaniu.

`DEFAULT_PERMISSION_CLASSES` - domyślna klasa permisji dla wszystkich widoków, warto zawsze mieć ją ustawiona na IsAuthenticated wtedy na pewno przez pomyłke nie damy ludzią dostępu do czegoś niezamierzonego. 

`DEFAULT_AUTHENTICATION_CLASSES` - W celu uwierzytelniania JWT tj. Json Web Token o którym wiecej juz niebawem musimy go zdefinować tutaj. SessionAuthentication oraz BasicAuthentication pozostawiamy ponieważ z tego bedzie korzystać panel admina Django. 

Zatemy skoro mamy przygotowany projekt czas brać się do pracy. W następnej sekcji zastanowimy się nad architektura oraz modelem bazy tak żeby aplikacja spełniała wymagania biznesowe.



