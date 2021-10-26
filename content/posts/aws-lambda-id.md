---
title: "Cara Deploy Python 3 di AWS Lambda"
date: 2021-10-24
draft: false
---
> Copied from Medium with slight correction

September 2018 lalu, saya mendapat kesempatan magang di kumparan sebagai Data Engineer. Sebagai tugas magang, saya ditugaskan membuat *distributed web scraper* untuk mengumpulkan data perhitungan Pemilu 2014 serta gambar scan C1 tiap TPS dari situs kawalpemilu.org.  
  
Kami memutuskan untuk menggunakan Amazon Web Services (AWS). *Worker* dari *distributed web scraper* nya dijalankan di AWS Lambda karena Lambda memungkinkan kami tidak repot di *maintenance*, dan dapat mendukung *concurrency* secara otomatis. Serta kami menggunakan AWS SQS untuk mengatur workernya.  
  
Sebagai pengingat, berikut panduan sederhana menjalankan sebuah program Python3 di AWS Lambda dalam Bahasa Indonesia. Beberapa hal dalam panduan ini dapat dilakukan menggunakan AWS CLI, namun demi kesederhanaan kita akan menggunakan AWS Console.  

# Peringatan
Cara ini terakhir dicoba pada *Desember 2018* lalu. Saat ini mungkin sudah terjadi perubahan cara deployment AWS Lambda.  
  
# Persyaratan
Dalam panduan ini saya berasumsi bahwa:
1. *Code* anda sudah menerapkan Function Handler.
2. *Code* anda sudah tercatat Git.
3. Anda memiliki akun AWS Console yang memiliki hak untuk membuat Lambda Function dan mengupload object ke S3.
4. AWS CLI terinstall dan ter-*setup* di komputer anda (ini opsional, hal-hal yang menggunakan AWS CLI dapat dilakukan melalui AWS Console).
5. Docker sudah terinstall di komputer anda.

# Bagian I: menyiapkan modul-modul
Pastikan semua modul yang digunakan project anda tercatat dalam sebuah `requirements.txt`. Apabila belum, buatlah `requirements.txt` dengan menjalankan command berikut dalam terminal:
```bash
pip3 freeze > requirements.txt
```  
pip akan mencatat semua nama dan versi package yang terinstall ke dalam sebuah file .txt (dalam panduan ini kita menggunakan pip3).  
  
Push file `requirements.txt` ke Git repository project anda:
```bash
git add requirements.txt
git commit -m "add requirements.txt"
git push
```  
  
Gunakan Docker untuk menjalankan image `dacut/amazon-linux-python-3.6`, AWS Lambda hanya dapat menggunakan modul yang diinstal menggunakan distribusi Linux tersebut:
```bash
docker run -it dacut/amazon-linux-python-3.6
```  
  
Pastikan anda sudah berada di environment docker container dengan menjalankan command `uname`. Outputnya akan seperti berikut:
>![Environment docker container](/aws-lambda-id/1.png#content)

Pada terminal docker container anda, install Git, lalu clone repository project anda:
```bash
# Install Git
yum install git

# Clone repository project anda
git clone <url-repository>

# masuk ke directory project anda
cd <nama-project>
```
Install package `virtualenv` menggunakan pip. virtualenv akan menampung semua modul yang akan dicompile ke dalam sebuah virtual environment.
```bash
pip3 install virtualenv
```
Setelah instalasi selesai, buat dan aktifkan sebuah virtual environment:
```bash
# buat sebuah virtual environment dengan nama 'env'
virtualenv env

# aktifkan virtual environment
. env/bin/activate
```
Setelah virtual environment anda aktif, install semua package yang terdaftar di requirements.txt:
```bash
pip3 install -r requirements.txt
```
Setelah proses instalasi selesai, kita akan copy semua modul yang terinstall di environment docker container ke environment komputer kita yang sebenarnya. sebelumnya kita perlu tahu `container_id` yang menjalankan image amazon-linux-python-3.6. Buka session terminal baru dan masukkan command berikut
```bash
docker container ls
```
Output command tersebut adalah sebagai berikut:
>![Output 'docker container ls'](/aws-lambda-id/2.png#content)

Copy container ID-nya, dan masukkan command berikut untuk meng-copy semua package yang di compile pada docker container ke environment komputer local kita. Package yang telah di compile berada pada directory *<nama-git-repo>/env/lib/python3.6/site-packages*
```bash
docker cp <CONTAINER_ID>:<nama-git-repo>/env/lib/python3.6/site-packages <LOCAL_PATH>
```
contoh:
>![Contoh docker cp](/aws-lambda-id/3.png#content)

# Bagian II: Deploy ke AWS Lambda
Buat sebuah folder baru, masukkan semua .py files dan modul-modul anda ke folder tersebut. lalu masuklah ke folder tersebut dan zip semua file yang ada di dalamnya
```bash
# buat sebuah folder baru
mkdir AWSLambdaPython3Tutorial

# masukkan semua modul-modul yang diinstall di docker container
cp -R site-packages/ AWSLambdaPython3Tutorial/

# masukkan semua .py files anda
cp -R <path-to-.py-files> AWSLambdaPython3Tutorial/

# masuk ke folder
cd AWSLambdaPython3Tutorial

# zip semua file yang ada di folder tersebut
zip -r AWSLambdaPython3Tutorial.zip .
```
Upload zip file tersebut ke S3. Anda dapat menggunakan AWS CLI atau upload melalui AWS Console.
```bash
aws s3 cp <path-to-.zip-file> s3://<bucket-S3-anda> --profile dhanangw-aws-es --profile <profile-aws-cli-name>
```
>![Upload melalui AWS Lambda Console](/aws-lambda-id/4.png#content)

kemudian copy S3 URL file .zip. Anda dapat temukan URL nya pada output dari command *aws s3 cp* atau anda dapat melihatnya AWS S3 Console.  
![Output 'aws s3 cp'](/aws-lambda-id/5.png#content)
  
>![S3 URL di AWS S3 Console](/aws-lambda-id/6.png#content)
  
Pada bagian “Function Code”, pilih “Upload a file from Amazon S3” pada drop-down “Code entry type”. paste copy URL S3 file .zip anda. Jangan lupa untuk mengubah pilihan “Runtime” menjadi Python
  
>![Copy-paste di field-field berikut](/aws-lambda-id/7.png#content)

# References
1. [Peter Begle's guide](https://medium.com/i-like-big-data-and-i-cannot-lie/how-to-create-an-aws-lambda-python-3-6-deployment-package-using-docker-d0e847207dd6)
2. [Joarley Moraes' guide](https://joarleymoraes.com/hassle-free-python-lambda-deployment/)
