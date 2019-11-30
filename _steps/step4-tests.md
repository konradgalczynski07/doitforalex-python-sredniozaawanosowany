---
layout: step
number: 4
title: Testy jednostkowe
permalink: step4/
---

Dobra luju dodamy tera testy i lecimy tutaj.

Masz i skopiuj testy:

```python
from django.test import TestCase
from django.contrib.auth import get_user_model
from django.urls import reverse

from rest_framework.test import APIClient
from rest_framework import status

REGISTER_USER_URL = reverse('users:register')
LOGIN_URL = reverse('users:login')
ME_URL = reverse('users:me')

User = get_user_model()


def create_sample_user(email='test@test.com', username='test', password='testpass'):
    return User.objects.create_user(email, username, password)


class PublicUserApiTests(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.payload = {
            'email': 'test@test.com',
            'username': 'test',
            'password': 'testpass',
        }

    def test_register_user_ok(self):
        res = self.client.post(REGISTER_USER_URL, self.payload)

        self.assertEqual(res.status_code, status.HTTP_201_CREATED)
        self.assertNotIn('password', res.data)

    def test_user_exists(self):
        self.client.post(REGISTER_USER_URL, self.payload)
        res = self.client.post(REGISTER_USER_URL, self.payload)

        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)

    def test_login_ok(self):
        user = create_sample_user()
        payload = {'username': user.username, 'password': 'testpass'}
        res = self.client.post(LOGIN_URL, payload)

        self.assertIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_200_OK)

    def test_login_invalid(self):
        payload = {'username': 'test', 'password': 'wrong'}
        res = self.client.post(LOGIN_URL, payload)

        self.assertNotIn('token', res.data)
        self.assertEqual(res.status_code, status.HTTP_400_BAD_REQUEST)


class PrivateUserApiTests(TestCase):
    def setUp(self):
        self.user = create_sample_user()
        self.client = APIClient()
        self.client.force_authenticate(user=self.user)

    def test_retrieve_user_unauthorized(self):
        res = APIClient().get(ME_URL)

        self.assertEqual(res.status_code, status.HTTP_401_UNAUTHORIZED)

    def test_retrieve_user_success(self):
        res = self.client.get(ME_URL)

        self.assertEqual(res.status_code, status.HTTP_200_OK)

    def test_post_me_not_allowed(self):
        res = self.client.post(ME_URL, {})

        self.assertEqual(res.status_code, status.HTTP_405_METHOD_NOT_ALLOWED)

    def test_update_user_profile(self):
        payload = {
            'username': 'changed',
            'fullname': 'new name',
            'password': 'newpassword123',
        }

        res = self.client.patch(ME_URL, payload)

        self.user.refresh_from_db()
        self.assertEqual(self.user.fullname, payload['fullname'])
        self.assertTrue(self.user.check_password(payload['password']))
        self.assertEqual(res.status_code, status.HTTP_200_OK)

```

