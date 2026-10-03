# Границы сервисов и эндпоинты

Три сервиса: **User Service**, **Content Service**, **Recommendation Service**.

## Разбивка таблиц по сервисам

 Сервис | Таблицы 

 **User Service** | `user`, `profile`, `follow`, `refresh_token`, `email_token`, `password_reset_token` 
 **Content Service** | `post`, `post_image`, `post_like`, `comment`, `comment_like`, `repost`, `hidden_post`, `post_report`,`post_favorite` 
 **Recommendation Service** | `post_text_embedding`, `post_image_embedding` 

## User Service

| Метод | Путь | Описание |

| POST | `/auth/register` | email, username, пароль, дата рождения → создаёт `user` (status=pending) + `profile`, отправляет код (`email_token`, purpose=verify) |
| POST | `/auth/register/confirm` | email + код → активирует аккаунт |
| POST | `/auth/register/resend-code` | email → новый `email_token` (verify), не чаще раза в минуту |
| POST | `/auth/login` | email/username + пароль → access-токен + refresh-токен |
| POST | `/auth/refresh` | refresh-токен → новая пара токенов (ротация, старый помечается `revoked_at`) |
| POST | `/auth/logout` | отзывает текущий refresh-токен |
| POST | `/auth/logout-all` | отзывает все refresh-токены пользователя |
| POST | `/auth/password/forgot` | email → код (`email_token`, purpose=reset) |
| POST | `/auth/password/forgot/verify-code` | email + код → проверяет код, выдаёт `password_reset_token` |
| POST | `/auth/password/reset` | reset-токен + новый пароль → меняет пароль, отзывает все refresh-токены |
| GET | `/users/me` | текущий пользователь (email, role, status) |
| GET | `/users/{username}` | публичный профиль: display_name, bio, avatar_url, счётчики, `is_following` |
| PATCH | `/users/me` | изменить display_name / bio / avatar_url |
| POST | `/users/{username}/follow` | подписаться |
| DELETE | `/users/{username}/follow` | отписаться |
| GET | `/users/{username}/followers` | список подписчиков, `cursor` + `limit` |
| GET | `/users/{username}/following` | список подписок, `cursor` + `limit` |
| GET | `/search/users` | поиск по `username`/`display_name`, `q` + `cursor` + `limit` |

## Content Service

| Метод | Путь | Описание |
| POST | `/posts` | multipart: `images[]` (1–10), `aspect_ratio`, `caption` → `post` (status=processing) + `post_image`, дёргает Recommendation Service на расчёт эмбеддинга |
| GET | `/posts/{id}` | пост целиком: изображения по `position`, подпись, счётчики, состояние лайка/избранного/репоста для текущего пользователя |
| POST | `/posts/batch` | `{ids: []}` → пачка постов (гидратация ленты «Для вас» по id от Recommendation Service и сетки поиска) |
| DELETE | `/posts/{id}` | удалить свой пост |
| GET | `/posts?author={username}` | посты автора для сетки профиля, `cursor` + `limit` (тем же эндпоинтом с `limit=3` — превью на ховер-карточке) |
| GET | `/posts/popular` | популярные посты за N дней — fallback-лента и пустой поиск, `days` + `limit` + `cursor` |
| GET | `/feed/following` | хронологическая лента по подпискам, `cursor` + `limit=10` |
| GET | `/search/posts` | сетка для пустого поиска (explore), `cursor` + `limit` |
| POST / DELETE | `/posts/{id}/like` | лайк / снять лайк |
| POST / DELETE | `/posts/{id}/favorite` | добавить / убрать из избранного |
| GET | `/favorites` | лента избранного текущего пользователя, `cursor` + `limit=10` |
| POST | `/posts/{id}/hide` | «не интересует» (запись в `hidden_post`) |
| POST | `/posts/{id}/report` | жалоба: `reason` + `description` |
| POST / DELETE | `/posts/{id}/repost` | репост / отмена репоста |
| GET | `/posts/{id}/comments` | комментарии верхнего уровня (`parent_comment_id IS NULL`), `cursor` + `limit=20` |
| GET | `/comments/{id}/replies` | ответы на комментарий, `cursor` + `limit` |
| POST | `/posts/{id}/comments` | `{text, parent_comment_id?}` — новый комментарий или ответ |
| DELETE | `/comments/{id}` | удалить комментарий (автор комментария или автор поста), каскадно удаляет ответы |
| POST / DELETE | `/comments/{id}/like` | лайк / снять лайк комментария |

## Recommendation Service

| Метод | Путь | Описание |
| GET | `/feed/for-you` | `user_id` + `cursor` + `limit=10` → ранжированный список `{post_id, score}` (холодный старт, ANN-поиск, ранжирование, fallback на `/posts/popular` из Content Service при нехватке данных или ошибке) |
| POST | `/feed/for-you/impressions` | `{user_id, post_ids: []}` — клиент сообщает, какие посты реально показаны, чтобы не повторять их в следующих выдачах |
| POST | `/embeddings/posts/{post_id}/compute` | вызывается Content Service при создании поста: считает CLIP+LM эмбеддинги, кладёт point в Qdrant, сохраняет `post_text_embedding` / `post_image_embedding` |
| GET | `/embeddings/posts/{post_id}` | статус и наличие эмбеддинга поста (служебный, для отладки) |

