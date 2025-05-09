# 09.02.06-patroni
# Тут происходит(ла) возня с моей дипломной работой: Модернизация PostgreSQL кластера на базе Patroni

# Что по сборке?
Нам будут нужны следующие пакеты:
  1. ETCD
  2. Patroni
  3. HAProxy
  4. PgBouncer

В работе используется 3 машины для самой работы и ещё одна для Ansible (одно из условий предприяти - автоматизация проекта)

Пока покажу мои мучения посреди ночного кодинга, как я исправлял что мне понаписала великая китайская нейросеть - дикпик (DeepSeek).
Заглядывая в будущее - она себя показала реально охуительно


Итак, поехали:

# Полезные фичи:
Генерация и заход в SSH по ключу без пароля

	ssh-keygen -t rsa -b 4096
	ssh-copy-id user@10.10.10.1
	ssh-copy-id user@10.10.10.2
	ssh-copy-id user@10.10.10.3

Фикс беспарольного sudo на виртуалках
	echo "user ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/user
	sudo chmod 440 /etc/sudoers.d/user

Полезные команды ETCD
	etcdctl endpoint status --write-out=table --cluster  (эта херь выводит табличку с кластером: лидеры, айдишники и т.д.)
	ETCDCTL_API=3 etcdctl --endpoints=http://10.10.10.1:2379,http://10.10.10.2:2379,http://10.10.10.3:2379 endpoint health    (эта херь показывает "жизнь" кластера)
	etcdctl member list

Полезные команды Patroni
	sudo -u postgres patronictl -c /etc/patroni.yml list    (выводит табличку с кластером: лидеры, реплики и т.д.)
	sudo systemctl reload postgresql    (перезагрузка постгрес)


ФИКСЫ ВСЕГО ЧТО АНСИБЛ УСТАНОВИЛ (ПАТРОНЫ ЕБУЧИЕ) (ЧТОБЫ НЕ ЗАБЫТЬ ИБО ЕБАЛ Я СРАЗУ ЭТО В АНСИБЛ ЗАНОСИТЬ)
	sudo cat /var/log/postgresql/postgresql-15-main.log | tail -50  (чекаем логи)
	PGPASSWORD="P@ssw0rd" psql -U postgres -h localhost -c "SELECT 1" (ставим пароль для юзера)
	vim /etc/postgresql/15/main/pg_hba.conf (Тут подключения или типа того, поставить надо, а зачем - уже не помню)
		local   all             postgres                                md5
	cp /var/lib/postgresql/15/main/postgresql.auto.conf /var/lib/postgresql/15/main/postgresql.conf (конфиг надо воткнуть ибо его нет)
	sudo systemctl reload postgresql (перезагружаем постгрес)
НА ВТОРОЙ НОДЕ ПРОБЛЕМЫ С КЛАСТЕРОМ
	May 08 21:55:17 node2 patroni[35512]: 2025-05-08 21:55:17,968 CRITICAL: system ID mismatch, node pg-node2 belongs to a different cluster: 7502229022341569694 != 7502> (дальше хуйня какая-то сорри не скопировал))
	May 08 21:55:18 node2 systemd[1]: patroni.service: Main process exited, code=exited, status=1/FAILURE
РЕШАЕТСЯ ТАК:
	sudo systemctl stop patroni									(тут всё ясно)
	sudo -u postgres rm -rf /var/lib/postgresql/15/main/*					(сносим нахуй)
	sudo systemctl start patroni									(после запуска кластер починится)
ДАЛЬШЕ ТАК, ПЕРВАЯ НОДА НЕ ЗАПУСТИЛАСЬ, А ВТОРАЯ ОСТАНОВЛЕНА
	root@node2:/home/user# sudo -u postgres patronictl -c /etc/patroni.yml list
	+ Cluster: pg-cluster (7502229022341569694) -----+----+-----------+
	| Member   | Host       | Role    | State        | TL | Lag in MB |
	+----------+------------+---------+--------------+----+-----------+
	| pg-node1 | 10.10.10.1 | Replica | start failed |    |   unknown |
	| pg-node2 | 10.10.10.2 | Replica | stopped      |    |   unknown |
	+----------+------------+---------+--------------+----+-----------+
РЕШАЕМ:
НА ПЕРВОЙ НОДЕ ПРОБЛЕМЫ С ПРАВАМИ:
	May 08 21:59:04 node1 patroni[36493]: 2025-05-09 01:59:04.360 GMT [36493] FATAL:  data directory "/var/lib/postgresql/15/main" has invalid permissions
	ДЕЛАЕМ ТАК И СПИНА НЕ БОЛИТ
	chmod 0750 /var/lib/postgresql/15/main
НОВАЯ ОШИБКА С КОНФИГ ФАЙЛОМ
	May 08 22:00:39 node1 patroni[36554]: 2025-05-09 02:00:39.276 GMT [36554] LOG:  could not open configuration file "/var/lib/postgresql/15/main/pg_hba.conf": No such >
	May 08 22:00:39 node1 patroni[36554]: 2025-05-09 02:00:39.276 GMT [36554] FATAL:  could not load pg_hba.conf
РЕШЕНИЕ:
	sudo -u postgres cp /etc/postgresql/15/main/pg_hba.conf /var/lib/postgresql/15/main/		(копируем конфиг)
	sudo chown postgres:postgres /var/lib/postgresql/15/main/pg_hba.conf					(меняем владельца и группу)
	sudo chmod 640 /var/lib/postgresql/15/main/pg_hba.conf							(ставим разрешения (rw-r----- если чо))

	sudo chown -R postgres:postgres /var/lib/postgresql/15/main/						(ну на всякий рекурсивно ещё валдельца хуйнём на папку)
	sudo find /var/lib/postgresql/15/main/ -type f -exec chmod 640 {} \;					(тут вроде проверка чего-то хуй занет, вроде можно и не втыкать в ансибл то)
	sudo find /var/lib/postgresql/15/main/ -type d -exec chmod 750 {} \;					(сейм стори)
ПРОБЛЕМА (СНОВА КОНФИГ ЕБАНЫЙ КАК Я ОБОЖАЮ pg_hba.conf)
	May 08 22:04:59 node1 patroni[37743]: 2025-05-08 22:04:59,169 INFO: no action. I am (pg-node1), the leader with the lock
	May 08 22:05:07 node1 patroni[37788]: 2025-05-09 02:05:07.390 GMT [37788] FATAL:  no pg_hba.conf entry for replication connection from host "10.10.10.2", user "replicator", no encryption
	May 08 22:05:09 node1 patroni[37743]: 2025-05-08 22:05:09,138 INFO: no action. I am (pg-node1), the leader with the lock
	May 08 22:05:12 node1 patroni[37789]: 2025-05-09 02:05:12.424 GMT [37789] FATAL:  no pg_hba.conf entry for replication connection from host "10.10.10.2", user "replicator", no encryption
РЕШЕНИЕ (vim /var/lib/postgresql/15/main/pg_hba.conf)
	host		replication	replicator		10.10.10.2/24		md5
ЗАМЕЧЕНА ОШИБКА НА ВТОРОЙ НОДЕ
	May 08 22:16:03 node2 patroni[35749]: pg_basebackup: error: connection to server at "10.10.10.1", port 5432 failed: FATAL:  password authentication failed for user ">
	May 08 22:16:03 node2 patroni[35749]: password retrieved from file "/tmp/pgpass"
	May 08 22:16:03 node2 patroni[35741]: 2025-05-08 22:16:03,223 ERROR: Error when fetching backup: pg_basebackup exited with code=1
РЕШЕНИЕ:
	sudo -u postgres psql -c "\du replicator" (проверка юзера на первой ноде)
ЕСЛИ НЕТ ЮЗЕРА (И ТАКОЕ БЫВАЕТ ОХУЕТЬ ДА ВЗЯЛ И ПРОПАЛ), ТО:
	sudo -u postgres psql -c "CREATE USER replicator WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'P@ssw0rd'"
ДАЛЬШЕ ТАКАЯ ХУЙНЯ НА ВТОРОЙ НОДЕ БЛЯТЬ ПОЯВИЛАСЬ
	May 08 22:22:51 node2 patroni[35919]: psycopg2.OperationalError: connection to server at "10.10.10.1", port 5432 failed: FATAL:  no pg_hba.conf entry for host "10.10.10.2", user "rewind_user", database "postgres", no encryption
	May 08 22:22:51 node2 patroni[35919]: 2025-05-08 22:22:51,223 INFO: no action. I am (pg-node2), a secondary, and following a leader (pg-node1)
РЕШЕНИЕ (vim /var/lib/postgresql/15/main/pg_hba.conf)
	host		postgres		rewind_user	10.10.10.2/24

НА ЭТОМ ВРОДЕ С ПАТРОНАМИ ПРОБЛЕМ НЕ БЫЛО БОЛЬШЕ!!!

root@node1:/home/user# sudo -u postgres patronictl -c /etc/patroni.yml list      #проверка кластера
+ Cluster: pg-cluster (7502229022341569694) --+----+-----------+
| Member   | Host       | Role    | State     | TL | Lag in MB |
+----------+------------+---------+-----------+----+-----------+
| pg-node1 | 10.10.10.1 | Leader  | running   | 10 |           |				#node1 - лидер
| pg-node2 | 10.10.10.2 | Replica | streaming | 10 |         0 |				#node2 - реплика
+----------+------------+---------+-----------+----+-----------+
root@node1:/home/user# systemctl restart patroni							#перезагружаем патроны
root@node1:/home/user# sudo -u postgres patronictl -c /etc/patroni.yml list		#проверка кластера
+ Cluster: pg-cluster (7502229022341569694) --+----+-----------+
| Member   | Host       | Role    | State     | TL | Lag in MB |
+----------+------------+---------+-----------+----+-----------+
| pg-node1 | 10.10.10.1 | Replica | streaming | 11 |         0 |				#реплика перешла к node1 потому что во время перезагрузки он был недоступен
| pg-node2 | 10.10.10.2 | Leader  | running   | 11 |           |				#node2 теперь не тварь дрожащая, а право имеет (лидер, ебать того в сраку)
+----------+------------+---------+-----------+----+-----------+



