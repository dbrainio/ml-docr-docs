### Installation

* Create **docker-compose.yml** file first:
```yamlex
version: "3"
services:
  classification:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      SERVICE: classification
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
      CUDA_VISIBLE_DEVICES: ${CUDA_VISIBLE_DEVICES}
      SERVICE: classifier

  multidocnet:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: ${CUDA_VISIBLE_DEVICES}
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
      CUDA_VISIBLE_DEVICES: ${CUDA_VISIBLE_DEVICES}
      SERVICE: ocr

  fieldnet:
    image: dbrainbinaries/docr:$VERSION
    environment:
      LICENSE: $LICENSE
      CUDA_VISIBLE_DEVICES: ${CUDA_VISIBLE_DEVICES}
      SERVICE: fieldnet
```
* Then create **.env** file with version, license & GPU usage info:
```.env
VERSION=v3.0.2
LICENSE=...
CUDA_VISIBLE_DEVICES=...
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

### Custom services names

Main services are **classification** & **recognition**. Both use other services.
If you want to customize hostnames, make your own images for both
**classification** & **recognition** services with modified config.
 
For instance:

```dockerfile
FROM dbrainbinaries/docr:v3.0.2

RUN sed -i 's/fieldnet:8080/custom-name:8080/g' configs/client.yml
```

Then build image and use it in docker-compose file instead of default one.

You can find all default hostnames in **configs/client.yml** file in default image.

For example:
```bash
$ docker run --rm --entrypoint="" -t dbrainbinaries/docr:v3.0.2 cat configs/client.yml | awk '/http:/{print $2}' 

http://multidocnet:8080
http://classifier:8080
http://fieldnet:8080
http://ocr:8080
http://heuristics:8080
```

---

### Cloud version

Cloud version is available using following URIs:
* https://classification.v3.dbrain.io
* https://recognition.v3.dbrain.io

The only difference in usage of cloud version vs docker image version is that
you have to add authorization header to each request. Like this:
```bash
curl -siX "POST" "https://classification.v3.dbrain.io/predict" \
-H 'Authorization: Token <API_TOKEN>' \
-H 'Content-Type: multipart/form-data; charset=utf-8' \
-H 'Accept: multipart/form-data' \
-F "image=@document.jpg"
```

Contact us if you want to get API token.

---

### Document types

* **bank_card**
* driver_license_card_back
* **driver_license_card_front**
* driver_license_plastic_back
* **driver_license_plastic_new_front**
* **driver_license_plastic_old_front**
* global_passport
* inn_organisation
* **inn_person**
* insurance_plastic
* military_id
* **ndfl2**
* not_document
* ogrn
* ogrnip
* other
* passport_blank_page
* passport_children
* passport_last_rf
* **passport_main**
* passport_main_handwritten
* passport_marriage
* passport_military
* passport_previous_docs
* passport_registration
* passport_zero_page
* pts_back
* pts_front
* registration_certificate
* snils_back
* **snils_front**
* **vehicle_registration_certificate_back**
* **vehicle_registration_certificate_front**
* **uzb_passport_main**

Only bolded types are supported for recognition.

---

### Fields for document types

Field names depending on doc type:

* bank_card
   * name
   * surname
   * number
   * month
   * year
* driver_license_card_front
   * date_of_birth
   * date_of_issue
   * name
   * series_number
   * surname
   * third_name
   * valid_before
* driver_license_plastic_new_front
   * date_of_birth
   * date_of_issue
   * name
   * series_number
   * surname
   * third_name
   * valid_before
* driver_license_plastic_old_front
   * date_of_birth
   * date_of_issue
   * name
   * series_number
   * surname
   * third_name
   * valid_before
* inn_person
   * date
   * fio
   * inn
   * issuer_number
   * number
   * series
* passport_main
   * date_of_birth
   * date_of_issue
   * first_name
   * issuing_authority
   * other_names
   * place_of_birth
   * series_and_number
   * sex
   * subdivision_code
   * surname
* snils_front
   * day_of_birth
   * month_of_birth
   * name
   * number
   * place_of_birth
   * sex
   * surname
   * third_name
   * year_of_birth
* vehicle_registration_certificate_front
   * car_body
   * chassis
   * color
   * engine_power
   * mass
   * max_mass
   * model_eng
   * model_rus
   * number_top
   * passport_number
   * passport_series
   * reg_number
   * release_year
   * series_top
   * type
   * vin
* vehicle_registration_certificate_back
   * country_code
   * date_of_issue
   * district
   * issuing_authority
   * name
   * number_bottom
   * number_top
   * series_bottom
   * series_top
   * state_eng
   * state_rus
   * street
   * surname
   * third_name
* ndfl2
   * agent
   * agent_inn
   * bottom
   * date_of_birth
   * doc_date
   * middle
   * name
   * other_name
   * ru_inn
   * surname
   * top_left_table
   * top_right_table
* uzb_passport_main
   * authority
   * date_of_birth
   * date_of_expiry
   * date_of_issue
   * gender
   * mrz_1
   * mrz_2
   * name
   * nation
   * number
   * place_of_birth
   * surname
