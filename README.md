# jenkins-old-project

Архів Jenkins-джобів 2021 року — навчальні приклади з курсу DevOps.

> Це експорт із `$JENKINS_HOME/jobs/<name>/`, не Jenkinsfile/Pipeline.
> Кожна тека — це окрема Freestyle Job (`config.xml` + історія білдів).
> Логи й changelog-и санітизовані: email замінено на `noreply@example.com`,
> публічну IP майстра — на `jenkins.local`.

## Що тут

| Джоб | Тип | Що робить |
|---|---|---|
| **Install-Apache2-fof-Debian-family-and-Deploy-web-page** | Freestyle | Клонує `web.git` (master), через **publish-over-ssh** заходить на ноду `jenkins-client`, ставить Apache (`apt install apache2`, `systemctl enable/start`, `ufw allow 80`), копіює сайт у `/var/www/html`, перезапускає сервіс. |
| **Auto-Deploy-web-page-from-github** | Freestyle | Той самий деплой на `/var/www/html`, але запускається автоматично через **ReverseBuildTrigger** після `Install-Apache2…` SUCCESS. Поллер відстежує `web.git` через GitHub webhook. |
| **Deploy-php-web-page** | Freestyle | Деплой PHP-сайту з `php-app-for-jenkins.git` у `/var/www/html` (теж через publish-over-ssh + рестарт Apache). |
| **Deploy-to-AWS-Elastic-Beanstalk** | Freestyle | Той самий PHP-репо, але деплой в **AWS Elastic Beanstalk** (application `php-app-for-jenkins`, env `Phpappforjenkins-env`). Тригериться GitHub webhook'ом. |
| **Install-qemu-guest-agent** | Freestyle | Інсталює `qemu-guest-agent` на віддалену ноду через SSH (без SCM, ручний trigger). |
| **Parametr** | Freestyle | Демо параметризованого білду: приймає `FOLDER_NAME` (string) і `uploaded_file` (file), виконує `ls -la $FOLDER_NAME` + `cat uploaded_file`. |

## Структура одного джоба

```
<JobName>/
├─ config.xml              # повна конфігурація Jenkins job
├─ nextBuildNumber         # лічильник наступного білду
├─ *-polling.log           # лог SCM-полінгу (webhook events)
└─ builds/
   ├─ 1/, 2/, ...          # історія білдів
   │  ├─ build.xml         # метадані запуску (тригер, статус, тривалість)
   │  ├─ log               # консольний лог білду
   │  └─ changelog.xml     # зміни в SCM між білдами
   ├─ legacyIds
   └─ permalinks
```

## Загальні елементи всіх джобів

- **Агент:** `<assignedNode>ubuntu</assignedNode>` — джоби виконувались на агенті з міткою `ubuntu`.
- **SCM credential:** `id_rsa` — посилання на запис у Jenkins credentials store (приватний ключ для GitHub SSH); сам ключ у репо НЕ зберігається.
- **AWS credential:** `jenkins-aws-beanstalk` — посилання на AWS credentials у Jenkins; самі ключі теж не в репо.
- **SSH target:** `jenkins-client` — назва SSH-сервера в **Publish over SSH** плагіні (Manage Jenkins → System → SSH servers); хост і ключ задавались у глобальній конфігурації Jenkins.
- **Log rotator:** збереження 5 останніх білдів.

## Як відновити в чистому Jenkins

1. Встановити необхідні плагіни: `git`, `publish-over-ssh`, `aws-beanstalk-publisher-plugin`, `parameterized-trigger`.
2. У **Manage Credentials** створити записи з такими ID:
   - `id_rsa` — SSH Username with private key для GitHub
   - `jenkins-aws-beanstalk` — AWS Access Key ID + Secret для Beanstalk
3. У **Manage Jenkins → System → SSH servers** додати сервер з назвою `jenkins-client` (хост, юзер, ключ).
4. Скопіювати теки джобів у `$JENKINS_HOME/jobs/` (наприклад: `cp -r Auto-Deploy-web-page-from-github $JENKINS_HOME/jobs/`).
5. **Manage Jenkins → Reload Configuration from Disk**.

## Залежності

- GitHub репозиторії, на які посилаються джоби (на 2021):
  - `rkuzmyn/web` — статичний сайт
  - `rkuzmyn/php-app-for-jenkins` — PHP-додаток
- AWS-акаунт з готовим Elastic Beanstalk-середовищем `Phpappforjenkins-env`.
- Linux-нода (`jenkins-client`) з sudo-доступом і SSH-сервером.

---

> 📚 Контекст: цей репо — навчальний артефакт із DevOps-курсу 2021 р.
> Сучасні Jenkins-проєкти варто писати як **declarative Pipeline** в `Jenkinsfile` поруч з кодом, а не Freestyle через UI.
