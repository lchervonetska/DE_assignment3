# DE_assignment3

## Projection of DWH

### Lineage Graph
<img width="1864" height="1515" alt="Untitled (5)" src="https://github.com/user-attachments/assets/3c9b08f7-4166-415b-8a87-9a37c37853b6" />

### DWH
<img width="3656" height="2808" alt="Untitled (4)" src="https://github.com/user-attachments/assets/561956d7-324b-4f33-aab8-0c734cb1cccb" />

## Staging layers

### stg_users
```sql
with source as (
    select * from {{ ref('users') }}
),
cleaned as (
    select
        user_id,
        trim(user_name) user_name,
        {{clean_text('email')}} as email,
        trim(country) as country,
        cast(signup_date as date)  as signup_date,
        cast(age as integer) as age,
        cast(is_active as boolean) as is_active
    from source
    where user_id is not null
      and age between 13 and 120
)
select * from cleaned
```

### stg_tracks
```sql
with source as (
    select * from {{ ref('tracks') }}
),

cleaned as (
    select
        track_id,
        trim(track_title) as track_title,
        artist_id,
        album_id,
        trim(genre) as genre,
        cast(duration_sec as integer) as duration_sec,
        cast(release_date as date) as release_date,
        cast(explicit as boolean) as explicit
    from source
    where track_id is not null
      and duration_sec > 0
)

select * from cleaned
```

### stg_subscriptions
```sql
with source as (
    select * from {{ ref('subscriptions') }}
),

cleaned as (
    select
        subscription_id,
        user_id,
        {{clean_text('plan_type')}} as plan_type,
        cast(start_date as date) as start_date,
        cast(end_date as date) as end_date,
        {{clean_text('status')}} as status,
        cast(auto_renew as boolean) as auto_renew
    from source
    where subscription_id is not null
      and user_id is not null
      and end_date >= start_date
)

select * from cleaned
```

### stg_playlists
```sql
with source as (
    select * from {{ ref('playlists') }}
),
cleaned as (
    select
        playlist_id,
        user_id,
        trim(playlist_name) as playlist_name,
        cast(created_at as timestamp) as created_at,
        cast(is_public as boolean) as is_public
    from source
    where playlist_id is not null
      and user_id is not null
)

select * from cleaned
```

### stg_payments
```sql
with source as (
    select * from {{ ref('payments') }}
),
cleaned as (
    select
        payment_id,
        user_id,
        subscription_id,
        cast(amount as decimal(10, 2)) as amount,
        cast(payment_date as timestamp) as payment_date,
        {{clean_text('payment_status')}} as payment_status
    from source
    where payment_id is not null
      and amount > 0
)

select * from cleaned
```

### stg_listening_events
```sql
with source as (
    select * from {{ ref('listening_events') }}
),
cleaned as (
    select
        event_id,
        user_id,
        track_id,
        cast(listened_at as timestamp) as listened_at,
        cast(seconds_played as integer) as seconds_played,
        {{ clean_text('device_type')}} as device_type
    from source
    where event_id is not null
      and user_id is not null
      and track_id is not null
      and seconds_played >= 0
)

select * from cleaned
```

### stg_likes
```sql
with source as (
    select * from {{ ref('likes') }}
),

cleaned as (
    select
        like_id,
        user_id,
        track_id,
        cast(liked_at as timestamp) as liked_at
    from source
    where like_id is not null
      and user_id is not null
      and track_id is not null
)

select * from cleaned
```

### stg_likes
```sql
with source as (
    select * from {{ ref('artists') }}
),
cleaned as (
    select
        artist_id,
        trim(artist_name) as artist_name,
        trim(country) as country,
        trim(genre) as genre,
        cast(monthly_listeners as integer) as monthly_listeners,
        cast(created_at as date) as created_at
    from source
    where artist_id is not null
)

select * from cleaned
```

### stg_artists
```sql
with source as (
    select * from {{ ref('artists') }}
),
cleaned as (
    select
        artist_id,
        trim(artist_name) as artist_name,
        trim(country) as country,
        trim(genre) as genre,
        cast(monthly_listeners as integer) as monthly_listeners,
        cast(created_at as date) as created_at
    from source
    where artist_id is not null
)

select * from cleaned
```

### stg_albums
```sql
with source as (
    select * from {{ ref('albums') }}
),

cleaned as (
    select
        album_id,
        artist_id,
        trim(album_title)  as album_title,
        cast(release_date as date) as release_date
    from source
    where album_id is not null
)

select * from cleaned
```

## dim tabels

### dim_artists
```sql
select * from {{ ref('stg_artists') }}
```

### dim_playlists
```sql
select * from {{ ref('stg_playlists') }}
```

### dim_subscriptions
```sql
select * from {{ ref('stg_subscriptions') }}
```

### dim_tracks
```sql
with albums as (
    select * from {{ ref('stg_albums') }}
),

tracks as (
    select * from {{ ref('stg_tracks') }}
),

artists as (
    select * from {{ ref('stg_artists') }}
),
joined as (
    select
        tracks.track_id,
        tracks.track_title,
        tracks.artist_id,
        artists.artist_name,
        albums.album_id,
        albums.album_title,
        artists.genre,
        tracks.duration_sec,
        tracks.release_date
    from tracks
    left join artists
    on tracks.artist_id = artists.artist_id
    left join albums
    on tracks.album_id = albums.album_id
)
select * from joined
```

### dim_users
```sql
select * from {{ ref('stg_users') }}
```

## fact tabels

### fct_likes
```sql
{{
    config(
        materialized='incremental',
        unique_key='like_id'
    )
}}

select * from {{ ref('stg_likes') }}

{% if is_incremental() %}
  where liked_at > (select max(liked_at) from {{ this }})
{% endif %}
```

### fct_listening_events
```sql
{{
    config(
        materialized='incremental',
        unique_key='event_id',
        incremental_strategy='delete+insert',
        incremental_predicates=[
            "listened_at >= date_trunc('day', current_timestamp - interval '3 days')"
        ]
    )
}}

select * from {{ ref('stg_listening_events') }}

{% if is_incremental() %}
  where listened_at >= (select max(listened_at) from {{ this }})
{% endif %}
```

### fct_payments
```sql
{{
    config(
        materialized='incremental',
        unique_key='payment_id'
    )
}}

select * from {{ ref('stg_payments') }}

{% if is_incremental() %}
  where payment_date > (select max(payment_date) from {{ this }})
{% endif %}
```

## mart layers

### mart_monthly_revenue
```sql
{{
    config(
        materialized='incremental',
        unique_key='year_month'
    )
}}
select
    strftime(payment_date, '%Y-%m')     as year_month,
    sum(amount)                         as total_revenue,
    count(payment_id)                   as total_payments,
    count(distinct user_id)             as unique_paying_users
from {{ ref('fct_payments') }}
group by year_month
```

### mart_subscription_status
```sql
with subscriptions as (
    select * from {{ ref('dim_subscriptions') }}
),

users as (
    select * from {{ ref('dim_users') }}
)

select
    users.user_id,
    users.user_name,
    subscriptions.plan_type,
    subscriptions.status,
    subscriptions.start_date,
    subscriptions.end_date,
    case
    when subscriptions.status = 'active'
    then 1
    else 0
end as is_active
from users
left join subscriptions
    on users.user_id = subscriptions.user_id
```

### mart_top_artists
```sql
with listening_events as (
    select
        artist_id,
        count(event_id) as total_listens
    from {{ ref('fct_listening_events') }}
    left join {{ ref('dim_tracks') }} using (track_id)
    group by artist_id
),

artists as (
    select
        artist_id,
        artist_name
    from {{ ref('dim_artists') }}
)

select
    artists.artist_id,
    artists.artist_name,
    coalesce(listening_events.total_listens, 0) as total_listens,
    rank() over (order by coalesce(listening_events.total_listens, 0) desc) as artist_rank
from artists
left join listening_events
    on artists.artist_id = listening_events.artist_id
```

### mart_top_tracks
```sql
with tracks as (
    select track_id, track_title from {{ ref('dim_tracks') }}
),

listening_stats as (
    select
        track_id,
        count(event_id) as listens_count
    from {{ ref('fct_listening_events') }}
    group by 1
)

select
    tracks.track_id,
    tracks.track_title,
    coalesce(listening_stats.listens_count, 0) as total_listens,
    rank() over (order by coalesce(listening_stats.listens_count, 0) desc) as track_rank
from tracks
left join listening_stats on tracks.track_id = listening_stats.track_id
```

### mart_track_engagement
```sql
with likes as (
    select
        track_id,
        count(like_id) as total_likes
    from {{ ref('fct_likes') }}
    group by track_id
),

listening_events as (
    select
        track_id,
        count(event_id) as total_listens
    from {{ ref('fct_listening_events') }}
    group by track_id
),

tracks as (
    select
        track_id,
        track_title
    from {{ ref('dim_tracks') }}
)
select
    tracks.track_id,
    coalesce(total_likes, 0) as total_likes,
    coalesce(total_listens, 0) as total_listens,
    coalesce(
        case
            when total_listens > 0 then (total_likes::float / total_listens)
            else 0
        end,
    0) as like_rate
from tracks
left join listening_events
    on tracks.track_id = listening_events.track_id
left join likes
    on tracks.track_id = likes.track_id
```

### mart_user_activity
```sql
{{
    config(
        materialized='incremental',
        unique_key='user_id'
    )
}}

with listening_events as (
    select * from {{ ref('fct_listening_events') }}
),

users as (
    select * from {{ ref('dim_users') }}
),

joined as (
    select
        users.user_id,
        users.user_name,
        listening_events.event_id,
        listening_events.track_id,
        listening_events.listened_at
    from users
    left join listening_events
    on users.user_id = listening_events.user_id
)
select user_id, user_name, count(event_id) as total_listens, count(distinct track_id) as unique_tracks,
max(listened_at) as last_listened_at, date_diff('day', max(listened_at), current_timestamp) AS days_since_last_activity
from joined

group by user_id, user_name
```

## dbt project
<img width="1275" height="652" alt="Screenshot 2026-03-31 225508" src="https://github.com/user-attachments/assets/c6cf6a85-5033-4cc1-9153-58ccd98955a5" />

## Analysis

### Top 10 tracks
<img width="1090" height="377" alt="Screenshot 2026-03-31 225747" src="https://github.com/user-attachments/assets/ef2701f6-7ab0-4d4f-b1ca-6d0dcce5871e" />

### Top 10 artists
<img width="1108" height="378" alt="Screenshot 2026-03-31 225839" src="https://github.com/user-attachments/assets/095764df-558f-4d03-8606-3570784db36d" />

### Monthly revenue
<img width="1243" height="724" alt="Screenshot 2026-03-31 230035" src="https://github.com/user-attachments/assets/8223531d-47cb-46f6-bb9d-478418cebffb" />

### Top 10 user activity
<img width="1517" height="378" alt="Screenshot 2026-03-31 230151" src="https://github.com/user-attachments/assets/989f7cc7-3a69-4378-828f-c242d4404838" />

### Top 10 track engagements
<img width="1070" height="388" alt="Screenshot 2026-03-31 230253" src="https://github.com/user-attachments/assets/9d90bb42-b86b-456f-af8e-e51c2f1a6a4c" />

### Users with active subscription
<img width="497" height="477" alt="Screenshot 2026-03-31 230322" src="https://github.com/user-attachments/assets/97490348-9804-45fe-a929-8c28e4a01ba6" />

