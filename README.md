### Table of Contents
1. [Versioning](#versioning)
1. [Documentation](#documentation)
1. [Cloud API version](#cloud-api-version)
1. [Installation](#installation)
1. [Services](#services)
1. [Healthcheck](#healthcheck)
1. [Classification](#classification)
1. [Recognition](#recognition)
1. [Classification + Recognition](#classification--recognition)
1. [Async version](#async-version)
1. [Document types & fields](#document-types--fields)

---

### Versioning

Versioning system is quite simple:
* Cloud API & Docker flavors of our product provide identical services for 
identical versions.
* Stable versions are labeled like this: v**X**.**Y**.**Z**, where **X** is 
major version, **Y** is minor version and **Z** is hotfix/enhacement version.
* Also there is special **latest** version - it can be very unstable, but 
includes all fresh enhancements & bugfixes. This version is halfway from one 
stable version to next stable one.

The most reasonable approach is to use any stable version of product, 
periodically making migration to newer versions.

But if you want to get full power of our solutions just try **latest** version. 

---

### Documentation

List of docs for stable versions:
* [v3.0.3](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.3)
* [v3.0.2](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.2)
* [v3.0.1](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.1)
* [v3.0.0](https://github.com/dbrainio/ml-docr-docs/tree/v3.0.0)

---

### Cloud API version

Cloud version is available using following URIs:
* https://classification.latest.dbrain.io
* https://recognition.latest.dbrain.io

The only difference in usage of cloud version vs docker image version is that
you have to add authorization header to each request. Like this:
```bash
curl -siX "POST" "https://classification.latest.dbrain.io/predict" \
-H 'Authorization: Token <API_TOKEN>' \
-H 'Content-Type: multipart/form-data; charset=utf-8' \
-H 'Accept: multipart/form-data' \
-F "image=@document.jpg"
```

[Contact us](https://docr.dbrain.io/) if you want to get API token.

---

### Installation

* Create **docker-compose.yml** file first:
```yamlex
version: "3"
services:
  web:
    image: dbrainbinaries/docr:$VERSION
    environment:
      SERVICE: web
      USERNAME: admin
      PASSWORD: Some_Passw0rd
      CLASSIFICATION_HOST: "$CLASSIFICATION_HOST"
      CLASSIFICATION_PASSPHRASE:
      RECOGNITION_HOST: "$RECOGNITION_HOST"
      RECOGNITION_PASSPHRASE:
      DATA: /data
    depends_on:
      - classification
      - recognition
    ports:
      - ${WEB_PORT:-8080}:8080

  classification:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      SERVICE: classification
      MULTIDOCNET_HOST: "$MULTIDOCNET_HOST"
      CLASSIFIER_HOST: "$CLASSIFIER_HOST"
      FIELDNET_HOST: "$FIELDNET_HOST"
      OCR_HOST: "$OCR_HOST"
      HEURISTICS_HOST: "$HEURISTICS_HOST"
    depends_on:
      - classifier
      - multidocnet
      - ocr
      - heuristics
    ports:
      - ${CLF_PORT:-23000}:8080

  recognition:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      SERVICE: recognition
      MULTIDOCNET_HOST: "$MULTIDOCNET_HOST"
      CLASSIFIER_HOST: "$CLASSIFIER_HOST"
      FIELDNET_HOST: "$FIELDNET_HOST"
      OCR_HOST: "$OCR_HOST"
      HEURISTICS_HOST: "$HEURISTICS_HOST"
    depends_on:
      - multidocnet
      - ocr
      - heuristics
      - fieldnet
    ports:
      - ${REC_PORT:-23001}:8080

  classifier:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: "$CUDA_VISIBLE_DEVICES"
      SERVICE: classifier

  multidocnet:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: "$CUDA_VISIBLE_DEVICES"
      SERVICE: multidocnet

  heuristics:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      SERVICE: heuristics

  ocr:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: "$CUDA_VISIBLE_DEVICES"
      SERVICE: ocr

  fieldnet:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: "$CUDA_VISIBLE_DEVICES"
      SERVICE: fieldnet
```
* Then create **.env** file with version, license & GPU usage info:
```.env
VERSION=latest
LICENSE=...
CUDA_VISIBLE_DEVICES=...

CLASSIFICATION_HOST=http://classification:8080
RECOGNITION_HOST=http://recognition:8080
MULTIDOCNET_HOST=http://multidocnet:8080
CLASSIFIER_HOST=http://classifier:8080
FIELDNET_HOST=http://fieldnet:8080
OCR_HOST=http://ocr:8080
HEURISTICS_HOST=http://heuristics:8080
```
* Possible values for VERSION variable: https://hub.docker.com/r/dbrainbinaries/docr/tags
* If want to run on CPU, leave CUDA_VISIBLE_DEVICES blank
* If want to run on GPU, try CUDA_VISIBLE_DEVICES=0 for example
* Then run services like this:
```bash
$  docker-compose up -d --force-recreate --build 
```

---

### Services

There are few types of services:
* **web**
  * optional web-demo
* **recognition**
  * main recognition endpoint
  * requires ~0.5Gb RAM
* **classification**
  * main classification endpoint
  * requires ~0.5Gb RAM
* **classifier**
  * requires ~1Gb RAM
  * requires ~1Gb GPU
* **multidocnet**
  * requires ~1Gb RAM
  * requires ~1Gb GPU
* **ocr**
  * requires ~1.5Gb RAM
  * requires ~1.5Gb GPU
* **heuristics**
  * requires ~0.5Gb RAM
* **fieldnet**
  * many subtypes inside
  * each subtype requires ~1Gb RAM
  * each subtype requires ~1Gb GPU

Note, all of these requirements are peak values, not average ones.

---

### Healthcheck

* Each service has following healthcheck endpoint:

```bash
$ curl -si 'http://127.0.0.1:23000/healthcheck'
```

* If **OK**:
 
```text
HTTP/1.1 204 No Content
Content-Length: 0
Content-Type: application/octet-stream
Date: Tue, 30 Jul 2019 11:01:43 GMT
Server: Python/3.6 aiohttp/3.5.4
```

---

### Classification:

* Run this to classify document:

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23000/predict' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: multipart/form-data' \
    -F 'image=@document.jpg'
```

* Multiple documents on single image are supported
* Supported formats of image:
  * JPEG
  * PNG
  * TIFF
  * PDF

* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: multipart/form-data;boundary=9b25e3de042e415ba384ba7f30b2185d
Transfer-Encoding: chunked
Date: Tue, 30 Jul 2019 11:32:40 GMT
Server: Python/3.6 aiohttp/3.5.4

--9b25e3de042e415ba384ba7f30b2185d
Content-Type: image/jpg
Content-Disposition: form-data; name="image-0"; filename="c2b1e58c-0622-40db-9a54-490a88adcfa2.jpg"
Content-Length: 258109

<CROPPED_DOCUMENT_IMAGE_BINARY_DATA>
--9b25e3de042e415ba384ba7f30b2185d
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name="text-0"
Content-Length: 15

{"type": "passport_main", "rotation": 1}
--9b25e3de042e415ba384ba7f30b2185d--

... MULTIPLE DOCUMENTS COULD BE FOUND ...

```

* Also **application/json** is allowed:

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23000/predict' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: application/json' \
    -F 'image=@document.jpg'
```

* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 344397
Date: Thu, 01 Aug 2019 08:26:26 GMT
Server: Python/3.6 aiohttp/3.5.4

{
    "items": [
        [
            {
                "crop": "data:image/jpg;base64,...",
                "document": {
                    "type": "passport_main",
                    "rotation": 1
                }
            },
            ... next documents ...
        ],
        ... next pages (could be greater 1 in case of PDF) ...
    ]
}
```

* Unicode characters in JSON are encoded as UTF-8

---

### Recognition:

* Run this to recognize document (use each document from classification step separately):

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: multipart/form-data' \
    -F 'image=@cropped_document_image_from_classification_step.jpg' \
    -F 'text=<DOCUMENT_TYPE_FROM_CLASSIFICATION_STEP>'
```

* Only single document on image is supported
* Supported formats of image:
  * JPEG
  * PNG
  * TIFF
  * PDF

* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: multipart/form-data;boundary=48612ba3cf8144fb835aeb033f14bbd8
Transfer-Encoding: chunked
Date: Tue, 30 Jul 2019 11:41:27 GMT
Server: Python/3.6 aiohttp/3.5.4

--48612ba3cf8144fb835aeb033f14bbd8
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name="text"
Content-Length: 1254

{
  "doc_type": "passport_main",
  "fields": {
    "date_of_birth": {
      "text": "18.03.1994",
      "confidence": 0.744497537612915
    },
    "date_of_issue": {
      "text": "14.01.2015",
      "confidence": 0.803318440914154
    },
    "first_name": {
      "text": "ВЯЧЕСЛАВ",
      "confidence": 0.9858060479164124
    },
    "issuing_authority": {
      "text": "ОТДЕЛОМ №2 УФМС РОССИИ ПО ***",
      "confidence": 0.9618614315986633
    },
    "other_names": {
      "text": "ЕВГЕНЬЕВИЧ",
      "confidence": 0.9819983243942261
    },
    "place_of_birth": {
      "text": "УСТЬ-ТАРКСКОГО С. УСТЬ-ТАРКА Р-НА НСВОСИБИРСКОЙ ОБЛ.",
      "confidence": 0.0
    },
    "series_and_number": {
      "text": "52** 40****",
      "confidence": 0.9477027654647827
    },
    "sex": {
      "text": "МУЖ.",
      "confidence": 0.6627151370048523
    },
    "subdivision_code": {
      "text": "550-***",
      "confidence": 0.9262790083885193
    },
    "surname": {
      "text": "П***",
      "confidence": 0.9620369076728821
    }
  }
}
--48612ba3cf8144fb835aeb033f14bbd8--
```

* Also **application/json** is allowed:

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: application/json' \
    -F 'image=@cropped_document_image_from_classification_step.jpg' \
    -F 'text=<DOCUMENT_TYPE_FROM_CLASSIFICATION_STEP>'
```

* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1846
Date: Thu, 01 Aug 2019 08:49:02 GMT
Server: Python/3.6 aiohttp/3.5.4

{
    "items": [
        {
            "doc_type": "passport_main",
            "fields": {
                "date_of_birth": {
                    "text": "18.03.1994",
                    "confidence": 0.7444975972175598
                },
                "date_of_issue": {
                    "text": "14.01.2015",
                    "confidence": 0.803318202495575
                },
                "first_name": {
                    "text": "В***",
                    "confidence": 0.9858060479164124
                },
                "issuing_authority": {
                    "text": "ОТДЕЛОМ ***",
                    "confidence": 0.9618613123893738
                },
                "other_names": {
                    "text": "Е***",
                    "confidence": 0.9819983243942261
                },
                "place_of_birth": {
                    "text": "УСТЬ-ТАРКСКОГО ***",
                    "confidence": 0.0
                },
                "series_and_number": {
                    "text": "5*** 408***",
                    "confidence": 0.9477028846740723
                },
                "sex": {
                    "text": "МУЖ.",
                    "confidence": 0.6627146005630493
                },
                "subdivision_code": {
                    "text": "550-***",
                    "confidence": 0.9262792468070984
                },
                "surname": {
                    "text": "П***",
                    "confidence": 0.9620369076728821
                }
            }
        }
    ]
}
```

* Here ranges for **confidence**:
  * [0.0, 0.4) - absolutely not sure
  * [0.4, 0.6] - "gray" zone
  * (0.6, 1.0] - pretty sure

* Unicode characters in JSON are encoded as UTF-8

---

### Classification + Recognition:

If you want to classify & recognize documents on image in single request, read this section.

* Run this to classify & recognize documents on image:

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict/classify/recognize' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: multipart/form-data' \
    -F 'image=@document.jpg' \
    -F 'text=<ALLOWED_DOCUMENT_TYPES>'
```

* Multiple documents on image are supported
* Supported formats of image:
  * JPEG
  * PNG
  * TIFF
  * PDF
* Example of **text** field: text=passport_main,inn_person
* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: multipart/form-data;boundary=48612ba3cf8144fb835aeb033f14bbd8
Transfer-Encoding: chunked
Date: Tue, 30 Jul 2019 11:41:27 GMT
Server: Python/3.6 aiohttp/3.5.4

--48612ba3cf8144fb835aeb033f14bbd8
Content-Type: text/plain; charset=utf-8
Content-Disposition: form-data; name="text"
Content-Length: 1254

{
  "doc_type": "passport_main",
  "fields": {
    "date_of_birth": {
      "text": "18.03.1994",
      "confidence": 0.744497537612915
    },
    "date_of_issue": {
      "text": "14.01.2015",
      "confidence": 0.803318440914154
    },
    "first_name": {
      "text": "ВЯЧЕСЛАВ",
      "confidence": 0.9858060479164124
    },
    "issuing_authority": {
      "text": "ОТДЕЛОМ №2 УФМС РОССИИ ПО ***",
      "confidence": 0.9618614315986633
    },
    "other_names": {
      "text": "ЕВГЕНЬЕВИЧ",
      "confidence": 0.9819983243942261
    },
    "place_of_birth": {
      "text": "УСТЬ-ТАРКСКОГО С. УСТЬ-ТАРКА Р-НА НСВОСИБИРСКОЙ ОБЛ.",
      "confidence": 0.0
    },
    "series_and_number": {
      "text": "52** 40****",
      "confidence": 0.9477027654647827
    },
    "sex": {
      "text": "МУЖ.",
      "confidence": 0.6627151370048523
    },
    "subdivision_code": {
      "text": "550-***",
      "confidence": 0.9262790083885193
    },
    "surname": {
      "text": "П***",
      "confidence": 0.9620369076728821
    }
  }
}
--48612ba3cf8144fb835aeb033f14bbd8--
```

* Also **application/json** is allowed:

```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict/classify/recognize' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: application/json' \
    -F 'image=@document.jpg' \
    -F 'text=<ALLOWED_DOCUMENT_TYPES>'
```

* If **OK**:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 1846
Date: Thu, 01 Aug 2019 08:49:02 GMT
Server: Python/3.6 aiohttp/3.5.4

{
    "items": [
        {
            "doc_type": "passport_main",
            "fields": {
                "date_of_birth": {
                    "text": "18.03.1994",
                    "confidence": 0.7444975972175598
                },
                "date_of_issue": {
                    "text": "14.01.2015",
                    "confidence": 0.803318202495575
                },
                "first_name": {
                    "text": "В***",
                    "confidence": 0.9858060479164124
                },
                "issuing_authority": {
                    "text": "ОТДЕЛОМ ***",
                    "confidence": 0.9618613123893738
                },
                "other_names": {
                    "text": "Е***",
                    "confidence": 0.9819983243942261
                },
                "place_of_birth": {
                    "text": "УСТЬ-ТАРКСКОГО ***",
                    "confidence": 0.0
                },
                "series_and_number": {
                    "text": "5*** 408***",
                    "confidence": 0.9477028846740723
                },
                "sex": {
                    "text": "МУЖ.",
                    "confidence": 0.6627146005630493
                },
                "subdivision_code": {
                    "text": "550-***",
                    "confidence": 0.9262792468070984
                },
                "surname": {
                    "text": "П***",
                    "confidence": 0.9620369076728821
                }
            }
        }
    ]
}
```

* Here ranges for **confidence**:
  * [0.0, 0.4) - absolutely not sure
  * [0.4, 0.6] - "gray" zone
  * (0.6, 1.0] - pretty sure

* Unicode characters in JSON are encoded as UTF-8

---

### Async version:

* Run this for **classification**:
```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23000/predict?async=true' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: (application/json OR multipart/form-data)' \
    -F 'image=@document.jpg'
```
* Run this for **recognition**:
```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict?async=true' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: (application/json OR multipart/form-data)' \
    -F 'image=@document.jpg'
    -F 'text=<DOCUMENT_TYPE>'
```
* Run this for **classification + recognition**:
```bash
$ curl -siX 'POST' \
    'http://127.0.0.1:23001/predict/classify/recognize?async=true' \
    -H 'Content-Type: multipart/form-data; charset=utf-8' \
    -H 'Accept: (application/json OR multipart/form-data)' \
    -F 'image=@document.jpg'
    -F 'text=<ALLOWED_DOCUMENT_TYPES>'
```

Example of response:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 57
Date: Fri, 24 May 2019 17:50:50 GMT
Server: Python/3.6 aiohttp/4.0.0a0

{
    "task_id": "e95064c3-a5e6-43a5-9fe9-34f54e8de6cc"
}
```

* To check result run this:
```bash
$ curl -si -H 'Accept: (application/json OR multipart/form-data)'\
    'http://127.0.0.1:(23000 OR 23001)/result/e95064c3-a5e6-43a5-9fe9-34f54e8de6cc'
```

If task was not found:

```
HTTP/1.1 404 Not Found
Content-Type: application/json; charset=utf-8
Content-Length: 97
Date: Fri, 24 May 2019 18:02:05 GMT
Server: Python/3.6 aiohttp/4.0.0a0

{
    "code": 404,
    "message": "Async task not found",
    "errno": 8,
    "traceback": null
}
```

If task was found, but it's not done yet:

```
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 96
Date: Fri, 24 May 2019 17:57:47 GMT
Server: Python/3.6 aiohttp/4.0.0a0

{
    "code": 200,
    "message": "Async task not done",
    "errno": 9,
    "traceback": null
}
```

If task was found & done, response is like regular response of sync version.

---

### Document types & fields

<table>
<thead><tr><th>Document type</th><th>Fields</th></tr></thead>
<tbody>
    <tr><td rowspan=5>bank_card</td><td>month</td></tr>
        <tr><td>name</td></tr>
        <tr><td>number</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>year</td></tr>
    <tr><td rowspan=7>driver_license_card_front</td><td>date_of_birth</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>name</td></tr>
        <tr><td>series_number</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
        <tr><td>valid_before</td></tr>
    <tr><td rowspan=7>driver_license_plastic_new_front</td><td>date_of_birth</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>name</td></tr>
        <tr><td>series_number</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
        <tr><td>valid_before</td></tr>
    <tr><td rowspan=7>driver_license_plastic_old_front</td><td>date_of_birth</td></tr>
        <tr><td>date_of_issue</td></tr>
        <tr><td>name</td></tr>
        <tr><td>series_number</td></tr>
        <tr><td>surname</td></tr>
        <tr><td>third_name</td></tr>
        <tr><td>valid_before</td></tr>
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
    <tr><td>driver_license_card_back</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>driver_license_plastic_back</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>global_passport</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>inn_organisation</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>insurance_plastic</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>military_id</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>not_document</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>ogrn</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>ogrnip</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>other</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_blank_page</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_children</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_last_rf</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_main_handwritten</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_marriage</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_military</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_previous_docs</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_registration</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>passport_zero_page</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>pts_back</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>pts_front</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>registration_certificate</td><td style="color: lightgrey;">not supported</td></tr>
    <tr><td>snils_back</td><td style="color: lightgrey;">not supported</td></tr>
</tbody>
</table>
