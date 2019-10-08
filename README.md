### Table of Contents
1. [Versioning](#versioning)
1. [Documentation](#documentation)
1. [Cloud API version](#cloud-api-version)
1. [Installation](#installation)
1. [Healthcheck](#healthcheck)
1. [General information](#general-information)
1. [Classification](#classification)
1. [Recognition](#recognition)
1. [Async version](#async-version)
1. [Document types & fields](#document-types--fields)

---

### Versioning

* v**X**.**Y**.**Z** (for instance, v3.0.5)
  * Stable versions.
  * **X** is a major version.
  * **Y** is a minor version.
  * **Z** is a patch version.
  * <span style="color:red">**Be careful!**</span> Backward compatibility between major versions is not guaranteed!
  * **Use in production.**
* **latest**
  * The most fresh & functional one, but can be unstable.
  * **Use it during integration period.**
* **experimental**
  * Includes all experimental features, absolutely unstable.
  * **Don't use it!**

---

### Documentation

* [experimental](https://github.com/dbrainio/ml-docr-docs/tree/experimental)
* [latest](https://github.com/dbrainio/ml-docr-docs/tree/latest)
* [v3.0.5](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.5)
* [v3.0.4](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.4)
* [v3.0.3](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.3)
* [v3.0.2](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.2)
* [v3.0.1](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.1)
* [v3.0.0](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.0)

---

### Cloud API version

* Web demo: https://experimental.dbrain.io
* Endpoints documentation: https://experimental.dbrain.io/docs
* Don't forget pass **Token** like this:
	```bash
	# token in header
	$ curl -siX POST \
		-H "Token: <API_TOKEN>" \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/classify"
	
	# token in query params
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/classify?token=<API_TOKEN>"
	```
* [Contact us](https://docr.dbrain.io/) if you want to get API token.

---

### Installation

* Create **docker-compose.yml** file first:

```yamlex
version: "3"
services:
  queue:
    image: rabbitmq:3.7-management-alpine

  redis:
    image: redis:5-alpine

  api:
    image: dbrainbinaries/docr:$VERSION
    env_file: &env .env
    environment:
      SERVICE: api
    ports:
      - ${API_PORT:-8080}:${API_PORT:-8080}

  client:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: client

  classifier:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: classifier

  multidocnet:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: multidocnet

  heuristics:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: heuristics

  ocr:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: ocr

  fieldnet:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: fieldnet

  crop_classifier:
    image: dbrainbinaries/docr:$VERSION
    env_file: *env
    environment:
      SERVICE: crop_classifier
```

* Then create **.env** file with settings:

```.env
# common
VERSION=experimental
LOG_LEVEL=ERROR

# cuda
CUDA_VISIBLE_DEVICES=...

# license
LICENSE=...

# redis
REDIS_URL=redis://redis

# queue
QUEUE_URL=amqp://guest:guest@queue

# api
API_HOST=0.0.0.0
API_PORT=8080
TASKS_LIFETIME=86400

# classifier
CLASSIFIER_BATCH_SIZE=4

# fieldnet
FIELDNET_BATCH_SIZE=4

# multidocnet
MULTIDOCNET_BATCH_SIZE=4
```

* Possible values for VERSION variable: https://hub.docker.com/r/dbrainbinaries/docr/tags
* If want to run on CPU, leave CUDA_VISIBLE_DEVICES blank
* If want to run on GPU, try CUDA_VISIBLE_DEVICES=0 for example
* Then run services like this:
	```bash
	$ docker-compose up -d --force-recreate --build \
		--scale classifier=1 \
		--scale multidocnet=1 \
		--scale heuristics=1 \
		--scale ocr=1 \
		--scale fieldnet=1 \
		--scale crop_classifier=1 \
		--scale client=1
	```

---

### Healthcheck

* Run command:
	```bash
	# docker version
	$ curl -si "http://127.0.0.1:8080/healthcheck"
	
	# cloud version
	$ curl -si "https://experimental.dbrain.io/healthcheck"
	```
* If **OK**:
	```text
	HTTP/2 200 
	content-type: application/json
	date: Mon, 07 Oct 2019 17:40:16 GMT
	server: uvicorn
	content-length: 16
	
	{"success":true}
	```

---

### General information

* Multiple documents on single image are supported
* Supported image formats:
  * JPEG
  * PNG
  * TIFF
  * BMP
  * GIF
  * PDF
* Unicode characters in JSON are encoded as UTF-8
* General response structure:
	```json
	{
		"detail": [...],
		"items": [...],  // optional field
		"task_id": {string|null}  // optional field
	}
	```
* Successful response:
	```text
	HTTP/2 200
	content-type: application/json
	date: Mon, 07 Oct 2019 17:46:06 GMT
	server: uvicorn
	content-length: 66
	
	{"detail":[],items:[...],task_id:null}
	```
* Response if license is not valid:
	```text
	HTTP/2 403
	content-type: application/json
	date: Mon, 07 Oct 2019 17:46:06 GMT
	server: uvicorn
	content-length: 66
	
	{"detail":[{"msg":"License is not valid","type":"license_error"}]}
	```
* Response if input params are not valid:
	```text
	HTTP/2 422 Unprocessable Entity
	date: Mon, 07 Oct 2019 17:59:32 GMT
	server: uvicorn
	content-length: 231
	content-type: application/json
	
	{"detail":[{"loc":[...],"msg":"...","type":"...","ctx":{...}}]}
	```
* If program error occurred:
	```text
	HTTP/2 500
	content-type: application/json
	date: Mon, 07 Oct 2019 17:41:36 GMT
	server: uvicorn
	content-length: 4142
	
	{"detail":[{"msg":"...","type":"..."}]}
	```

---

### Classification:

* Run command:
	```bash
	# docker version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"http://127.0.0.1:8080/classify"
	
	# cloud version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/classify"
	```
* If **OK**:
	```text
	HTTP/2 200
	content-type: application/json
	date: Mon, 07 Oct 2019 17:33:37 GMT
	server: uvicorn
	content-length: 485833
	
	{"detail":[],"items":[{"document":{"type":"passport_main","rotation":1},"crop":"data:image/jpg;base64,..."}],"task_id":null}
	```

---

### Recognition:

* Run command:
	```bash
	# docker version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"http://127.0.0.1:8080/recognize?doc_type=passport_main"
	
	# cloud version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/recognize?doc_type=passport_main"
	```
* Use **doc_type=...** get parameter multiple times for each doc type you are going to recognize on image 
* If **OK**:
	```text
	HTTP/2 200
	date: Mon, 07 Oct 2019 22:59:59 GMT
	server: uvicorn
	content-length: 1565
	content-type: application/json
	
	{"detail":[],"items":[{"doc_type":"...","fields":{"...":{"text":"...","confidence":0.123},...}},...],"task_id":null}
	```
* Meaning of **confidence**:
  * [0.0, 0.4) - absolutely not sure
  * [0.4, 0.6] - "gray" zone
  * (0.6, 1.0] - pretty sure

---

### Async version:

* Just add **async=true** get parameter to request:
	```bash
	# classification, docker version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"http://127.0.0.1:8080/classify?async=true"
	
	# classification, cloud version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/classify?async=true"

	# recognition, docker version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"http://127.0.0.1:8080/recognize?doc_type=passport_main&async=true"
	
	# recognition, cloud version
	$ curl -siX POST \
		-F "image=@document.jpg" \
		"https://experimental.dbrain.io/recognize?doc_type=passport_main&async=true"
	```
* If **OK**:
	```text
	HTTP/2 200
	date: Mon, 07 Oct 2019 23:34:17 GMT
	server: uvicorn
	content-length: 73
	content-type: application/json
	
	{"detail":[],"items":[],"task_id":"a824bd7d-788f-4b42-9f28-5d2d54db178f"}
	```
* Check results:
	```bash
	# docker version
	$ curl -si "http://127.0.0.1:8080/result/a824bd7d-788f-4b42-9f28-5d2d54db178f"
	
	# cloud version
	$ curl -si "https://experimental.dbrain.io/result/a824bd7d-788f-4b42-9f28-5d2d54db178f"
	```
* If task was not found:
	```text
	HTTP/2 404
	date: Mon, 07 Oct 2019 23:41:41 GMT
	server: uvicorn
	content-length: 65
	content-type: application/json
	
	{"detail":[{"msg":"Async task not found","type":"result_error"}]}
	```
* If task was found, but it's not done yet:
	```text
	HTTP/2 425
	date: Mon, 07 Oct 2019 23:41:41 GMT
	server: uvicorn
	content-length: 64
	content-type: application/json
	
	{"detail":[{"msg":"Async task not done","type":"result_error"}]}
	```
* If task was found & done, response is like regular response of sync version.

---

### Document types & fields

<table>
<thead><tr><th>Document type</th><th>Fields</th></tr></thead>
<tbody>
    <tr><td rowspan=4>bank_card</td><td>cardholder_name</td></tr>
        <tr><td>month</td></tr>
        <tr><td>number</td></tr>
        <tr><td>year</td></tr>
    <tr><td rowspan=7>driver_license_card_front</td><td>date_of_birth</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>name</td></tr>
        <tr><td>series_number</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
        <tr><td>valid_before</td></tr>
    <tr><td rowspan=10>driver_license_plastic_new_front</td><td>field_date_end</td></tr>
        <tr><td>field_date_from</td></tr>
        <tr><td>field_date_of_birth</td></tr>
        <tr><td>field_issuer</td></tr>
        <tr><td>field_name</td></tr>
        <tr><td>field_number</td></tr>
        <tr><td>field_patronymic</td></tr>
        <tr><td>field_place_of_birth</td></tr>
        <tr><td>field_place_of_issue</td></tr>
        <tr><td>field_surname</td></tr>
    <tr><td rowspan=12>driver_license_plastic_old_front</td><td>field_date_end</td></tr>
        <tr><td>field_date_from</td></tr>
        <tr><td>field_date_of_birth</td></tr>
        <tr><td>field_doc_number</td></tr>
        <tr><td>field_doc_series</td></tr>
        <tr><td>field_issuer</td></tr>
        <tr><td>field_name</td></tr>
        <tr><td>field_patronymic</td></tr>
        <tr><td>field_place_of_birth</td></tr>
        <tr><td>field_place_of_issue</td></tr>
        <tr><td>field_special</td></tr>
        <tr><td>field_surname</td></tr>
    <tr><td rowspan=6>inn_person</td><td>date</td></tr>
        <tr><td>fio</td></tr>
        <tr><td>inn</td></tr>
        <tr><td>issuer_number</td></tr>
        <tr><td>number</td></tr>
        <tr><td>series</td></tr>
    <tr><td rowspan=12>ndfl2</td><td>agent</td></tr>
        <tr><td>agent_inn</td></tr>
        <tr><td>bottom</td></tr>
        <tr><td>date_of_birth</td></tr>
        <tr><td>doc_date</td></tr>
        <tr><td>middle</td></tr>
        <tr><td>name</td></tr>
        <tr><td>other_name</td></tr>
        <tr><td>ru_inn</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>top_left_table</td></tr>
        <tr><td>top_right_table</td></tr>
    <tr><td rowspan=13>kgz_passport_main</td><td>date_of_birth</td></tr>
        <tr><td>date_of_expiry</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>issuer</td></tr>
        <tr><td>mrz_1</td></tr>
        <tr><td>mrz_2</td></tr>
        <tr><td>name</td></tr>
        <tr><td>nation</td></tr>
        <tr><td>passport_num</td></tr>
        <tr><td>personal_number</td></tr>
        <tr><td>place_of_birth</td></tr>
        <tr><td>sex</td></tr>
        <tr><td>surname</td></tr>
    <tr><td rowspan=21>passport_main</td><td>date_of_birth</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>first_name</td></tr>
        <tr><td>issuing_authority</td></tr>
        <tr><td>other_names</td></tr>
        <tr><td>place_of_birth</td></tr>
        <tr><td>series_and_number</td></tr>
        <tr><td>sex</td></tr>
        <tr><td>subdivision_code</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>mrz_checksum</td></tr>
        <tr><td>mrz_code</td></tr>
        <tr><td>mrz_country</td></tr>
        <tr><td>mrz_date_of_birth</td></tr>
        <tr><td>mrz_date_of_issue</td></tr>
        <tr><td>mrz_name</td></tr>
        <tr><td>mrz_nationality</td></tr>
        <tr><td>mrz_number</td></tr>
        <tr><td>mrz_other_name</td></tr>
        <tr><td>mrz_sex</td></tr>
        <tr><td>mrz_surname</td></tr>
    <tr><td rowspan=10>tjk_passport_main</td><td>date_of_birth</td></tr>
        <tr><td>date_of_expiry</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>mrz_1</td></tr>
        <tr><td>mrz_2</td></tr>
        <tr><td>name</td></tr>
        <tr><td>nation</td></tr>
        <tr><td>number</td></tr>
        <tr><td>sex</td></tr>
        <tr><td>surname</td></tr>
    <tr><td rowspan=4>pts_back</td><td>address</td></tr>
        <tr><td>date</td></tr>
        <tr><td>name</td></tr>
        <tr><td>special_marks</td></tr>
    <tr><td rowspan=19>pts_front</td><td>1</td></tr>
        <tr><td>10</td></tr>
        <tr><td>11</td></tr>
        <tr><td>14</td></tr>
        <tr><td>15</td></tr>
        <tr><td>18</td></tr>
        <tr><td>2</td></tr>
        <tr><td>3</td></tr>
        <tr><td>4</td></tr>
        <tr><td>5</td></tr>
        <tr><td>6</td></tr>
        <tr><td>7</td></tr>
        <tr><td>8</td></tr>
        <tr><td>9</td></tr>
        <tr><td>address</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>doc_number</td></tr>
        <tr><td>issuer</td></tr>
        <tr><td>special_marks</td></tr>
    <tr><td rowspan=12>snils_front</td><td>day_of_birth</td></tr>
        <tr><td>day_of_issue</td></tr>
        <tr><td>month_of_birth</td></tr>
        <tr><td>month_of_issue</td></tr>
        <tr><td>name</td></tr>
        <tr><td>number</td></tr>
        <tr><td>place_of_birth</td></tr>
        <tr><td>sex</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
        <tr><td>year_of_birth</td></tr>
        <tr><td>year_of_issue</td></tr>
    <tr><td rowspan=10>uzb_passport_main</td><td>authority</td></tr>
        <tr><td>date_of_birth</td></tr>
        <tr><td>date_of_expiry</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>gender</td></tr>
        <tr><td>name</td></tr>
        <tr><td>nation</td></tr>
        <tr><td>number</td></tr>
        <tr><td>place_of_birth</td></tr>
        <tr><td>surname</td></tr>
    <tr><td rowspan=18>vehicle_registration_certificate_back</td><td>building</td></tr>
        <tr><td>building_number</td></tr>
        <tr><td>city</td></tr>
        <tr><td>country_code</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>district</td></tr>
        <tr><td>home</td></tr>
        <tr><td>issuing_authority</td></tr>
        <tr><td>name</td></tr>
        <tr><td>number_bottom</td></tr>
        <tr><td>number_top</td></tr>
        <tr><td>series_bottom</td></tr>
        <tr><td>series_top</td></tr>
        <tr><td>state_eng</td></tr>
        <tr><td>state_rus</td></tr>
        <tr><td>street</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
    <tr><td rowspan=20>vehicle_registration_certificate_front</td><td>car_body</td></tr>
        <tr><td>category</td></tr>
        <tr><td>chassis</td></tr>
        <tr><td>color</td></tr>
        <tr><td>ecologic_class</td></tr>
        <tr><td>engine_power</td></tr>
        <tr><td>mass</td></tr>
        <tr><td>max_mass</td></tr>
        <tr><td>model_eng</td></tr>
        <tr><td>model_rus</td></tr>
        <tr><td>number_bottom</td></tr>
        <tr><td>number_top</td></tr>
        <tr><td>passport_number</td></tr>
        <tr><td>passport_series</td></tr>
        <tr><td>reg_number</td></tr>
        <tr><td>release_year</td></tr>
        <tr><td>series_bottom</td></tr>
        <tr><td>series_top</td></tr>
        <tr><td>type</td></tr>
        <tr><td>vin</td></tr>
    <tr><td>driver_license_card_back</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>driver_license_plastic_back</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>global_passport</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>inn_organisation</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>insurance_plastic</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>military_id</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>not_document</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>ogrn</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>ogrnip</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>other</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_blank_page</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_children</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>kgz_passport_plastic</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_last_rf</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_main_handwritten</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_marriage</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_military</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_previous_docs</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_registration</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>tjk_passport_other</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_uzbek_main_page</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>passport_zero_page</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>registration_certificate</td><td style="color: lightgrey;">classification only</td></tr>
    <tr><td>snils_back</td><td style="color: lightgrey;">classification only</td></tr>
</tbody>
</table>
