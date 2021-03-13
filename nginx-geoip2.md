Ограничение по странам доступа к отдельным частям сайта

Бывает так, что нужно ограничить доступ по странам, например с целью уменьшения попыток взлома ограничить доступ к wp-login при использовании WP только той страной откуда все работают с сайтом

Для этих целей мы используем модуль geoip2 для nginx и GeoIPLite базу от MaxMind.
Скрипт для скачки и обновления баз можно взять [тут](https://github.com/Nill-R/scripts/blob/master/geoiplite_update.bash), скрипт требует ключа, который можно получить бесплатно зарегистрировавшись на сайте MaxMind.

У nginx'а нам понадобится модуль ngx_http_geoip2, проверить собран ли с ним ваш nginx можно командой `nginx -V|grep geoip2`, если включен, то можно перейти к настройке. Создаем файл `geoip2.conf` в `/etc/nginx/conf.d/` со следующим содержимым
```
geoip2 /var/lib/GeoIP/city.mmdb {
    auto_reload 1440m;
    $geoip2_metadata_city_build metadata build_epoch;
    $geoip2_data_city_name city names en;
    $geoip2_data_city_geonameid city geoname_id;
    $geoip2_data_continent_code continent code;
    $geoip2_data_continent_geonameid continent geoname_id;
    $geoip2_data_continent_name continent names en;
    $geoip2_data_country_geonameid country geoname_id;
    $geoip2_data_country_code country iso_code;
    $geoip2_data_country_name country names en;
    $geoip2_data_country_is_eu country is_in_european_union;
    $geoip2_data_location_accuracyradius location accuracy_radius;
    $geoip2_data_location_latitude location latitude;
    $geoip2_data_location_longitude location longitude;
    $geoip2_data_location_metrocode location metro_code;
    $geoip2_data_location_timezone location time_zone;
    $geoip2_data_postal_code postal code;
    $geoip2_data_rcountry_geonameid registered_country geoname_id;
    $geoip2_data_rcountry_iso registered_country iso_code;
    $geoip2_data_rcountry_name registered_country names en;
    $geoip2_data_rcountry_is_eu registered_country is_in_european_union;
    $geoip2_data_region_geonameid subdivisions 0 geoname_id;
    $geoip2_data_region_iso subdivisions 0 iso_code;
    $geoip2_data_region_name subdivisions 0 names en;
  }

map $geoip2_data_country_code $allowed_country {
        default no;
        RU yes;
}
```

Путь к файлу базы прописываем такой, как прописали в скрипте скачивающем и обновляющим базы, в map прописываем страны доступ из которых нам надо разрешить(в моем примере это RU)
Проверяем в `nginx.conf`, что у нас файлы из `/etc/nginx/conf.d` инклюдятся в него, если да, то делаем `nginx -t`, что бы проверить, что конфиг валидный и если все хорошо, то переходим к конфигу по самому домену.

Как я уже сказал мне нужно разрешить доступ к `wp-login.php` ограничившись конкретной страной, а потому добавляем

```
location ~ ^/wp-login.php {
        if ($allowed_country = no) {
            return 451;
        }
}
```
Если переменная `$allowed_country` имеет значение `no`, то отдаем ошибку 451, в противном случае пропускаем. Ошибку можно задать на свой вкус, можно отдавать хоть 403, хоть 418, хоть 404, тут дело в личном вкусе и фантазии, я посчитал, что 451 для данного случая самое уместное

Снова делаем `nginx -t`, для проверки конфига, если все ок, то перезапускаем nginx командой `systemctl restart nginx` и наслаждаемся результатом. При доступе из РФ все проходит нормально, при попытке доступа из других стран(например через VPN для проверки) получаем 451. Таким образом мы(в данном конкретном случае) уменьшаем на порядок количество ботов которые пытаются подолбится об `wp-login`, так как большинство идут из Китая и Бразилии, а потому отваливаются на уровне проверки страны и до странички не доходят

В принципе можно ограничить и более жестко по городам, так как я использую базу с городами, но российские провайдеры очень часто к полю города относятся наплевательски и, к примеру, выходя с мобильного в Кёнигсберге по GeoIP я всегда считаюсь находящимся в Санкт-Петербурге. По этой причине я предпочитаю использовать ограничение по странам.
