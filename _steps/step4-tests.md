---
layout: step
number: 4
title: Testy jednostkowe
permalink: step4/
---

Testy jednostkowe to w zasadzie kod testujący inny kod. Aby utrzymać wysokie standardy tworzonego oprogramowania i mieć pewność że nic nie zepsujemy dodając zmiany w przyszłości warto je pisać.

W naszych testach skorzystamy z klas dostarczonych przez Django i DRF `TestCase` oraz `APIClient`. Do endpointów dostaniemy się za pomoca funkcji `reverse` której podamy nazwę URL'a. Bedzię to wygladało tak `reverse('nazwa_aplikacji:nazwa_scieżki')`. Ponadto napiszemy funkcje pomocniczą która będzie tworzyła nowego użytkownika.


```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from django.urls import reverse

from rest_framework.test import APIClient
from rest_framework import status

REGISTER_USER_URL = reverse('users:register')
LOGIN_URL = reverse('users:login')

User = get_user_model()


# helper
def create_sample_user(email='test@test.com', username='test', password='testpass'):
    return User.objects.create_user(email, username, password)
```

Gdy mamy potrzebne importy, url'e i funkcje pomocniczą przejdźmy do klasy testowej oraz pierwszego testu. W klasie tej jako pierwszą dodamy metodę `setUp`, wrzucamy tam wszystko co będzie przydane w testach gdyż każdy test bedzie miał potem do tego dostęp. W naszym wypadku dodamy tam instancje klasy `APIClient`, umożliwi nam ona na wysyłanie requestów oraz `payload` z danymi testowego użytkownika.


```python
class PublicUserApiTests(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.payload = {
            'email': 'test@test.com',
            'username': 'test',
            'password': 'testpass',
        }

    def test_register_user_ok(self):
        response = self.client.post(REGISTER_USER_URL, self.payload)

        self.assertEqual(response.status_code, status.HTTP_201_CREATED)
        self.assertNotIn('password', response.data)
```

W pierwszym tescie sprawdzamy endpoint do rejestrowania się, korzystając z instancji `APIClient` wysyłamy POST'em payload na podany URL i przypisujemy to do zmiennej `response`. Następnie asercją sprawdzamy czy kod statusu pasuję do oczekiwanego przez nas 201 i czy hasło nie znajduję się w zwracanych danych.

Testy Django odpalamy przy pomocy:

```
python instagram-backend/manage.py test users
```

Napiszmy kolejny test który będzie sprawdzał czy da się stworzyć użytkownika z takimi samymi danymi dwa razy.

```python
    def test_user_exists(self):
        self.client.post(REGISTER_USER_URL, self.payload)
        response = self.client.post(REGISTER_USER_URL, self.payload)

        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

Ponownie wysyłamy POST'em dane na endpoint rejestracji. Robimy to dwa razy a drugi request przypisujemy do zmiennej żeby następnie sprawdzić czy kod HTTP zgadza się z oczekiwaną 400.

Ok, dodajmy jeszcze może test sprawdzający poprawne logowanie.

```python
    def test_login_ok(self):
        user = create_sample_user()
        payload = {'username': user.username, 'password': 'testpass'}
        response = self.client.post(LOGIN_URL, payload)

        self.assertIn('token', response.data)
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```

Tutaj najpierw za pomocą funkcji pomocniczej tworzymy nowego użytkownika a potem używając jego danych próbujemy się zalogować. Przeprowadzamy asercję czy `token` znajduję się w zwracanych danych oraz czy kod HTTP to oczekiwane 200_OK.

Ok, czy łapiesz już działanie testów? Napisz teraz swoj własny test o nazwie `test_login_invalid` sprawdzający logowanie się nieistniejącym użytkownikiem.

```python
    def test_login_invalid(self):
        payload = {'username': 'test', 'password': 'wrong'}
        response = self.client.post(LOGIN_URL, payload)

        self.assertNotIn('token', response.data)
        self.assertEqual(response.status_code, status.HTTP_400_BAD_REQUEST)
```

Chciałbyś wiecej? Proszę bardzo - napisz test sprawdzający rejestracje ze zbyt krótkim loginem a następnie ze zbyt krótkim hasłem.

Przydałoby się napisać jeszcze testy do prywatnych endpointów czyli tych które wymagają uwierzytelniania. Spójrzmy na poniższe przykłady:

```python

class PrivateUserApiTests(TestCase):
    def setUp(self):
        self.user = create_sample_user()
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_retrieve_user_unauthorized(self):
        response = APIClient().get(ME_URL)

        self.assertEqual(response.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_retrieve_user_success(self):
        response = self.client.get(ME_URL)

        self.assertEqual(response.status_code, status.HTTP_200_OK)

    def test_update_user_profile(self):
        payload = {
            'username': 'changed',
            'fullname': 'new name',
            'password': 'newpassword123',
        }

        response = self.client.patch(ME_URL, payload)

        self.user.refresh_from_db()
        self.assertEqual(self.user.fullname, payload['fullname'])
        self.assertTrue(self.user.check_password(payload['password']))
        self.assertEqual(response.status_code, status.HTTP_200_OK)
```