---
title: "Распаковка архивов на лету в Java"
date: 2024-02-11
---

# Проблема

Классический подход при работе с архивами в программах состоит в следующем:

1. Загрузить архив с удаленного ресурса
2. Распаковать архив средствами нужной библиотеки
3. Удалить файл архива

К такому предельно простому подходу уже все давно привыкли и он не вызывает
каких-либо вопросов. Проблемы возникают тогда, когда либо архив становится
очень большим, и следовательно процесс загрузки занимает большое количество
времени, увеличивая тем самым время простоя CPU в ожидании окончания процессов
ввода/вывода, либо архивов много и распаковка каждого из них тоже занимает
большое количество времени. Кроме того, под пунктом 3 дополнительная операция
удаления архива после распаковки, что тоже отнимает определенное время если
архив большой.

Казалось бы, ничего не поделать. Придется выделять на каждый архив отдельный
поток, который будет последовательно выполнять данные шаги. Или же выделять
отдельные потоки на загрузку и на распаковку.

Но что если каким-то образом объединить процесс загрузки и распаковки? Ведь
любой ZIP архив по сути представляет собой последовательный набор заголовков
файлов (local file header) и их содержимого. Если дождаться получения
заголовка первого файла и его содержимого, то не достигнув конца архива мы
вполне можем распаковать и записать его на диск, затем продолжив процесс
загрузки и проделав то же самое с последующими файлами. 

Плюсы такого подхода очевидны: 

- Оптимизация ввода/вывода. Загружая весь архив на диск мы делаем запись
  большого потока данных, а в процессе расжатия такую же большую (в зависимости
  от размера самого архива) операцию чтения и записи содержимого отдельных
  файлов которые в нем хранятся. В то же время расжимая его на лету в процессе
  загрузки мы сразу пишем содержимые в нем файлы н диск без промежуточной
  записи и чтения самого из архива на диске.
- Не нужно будет удалять сам файл архива после загрузки. 
- Отдельные файлы могут быть просмотрены ещё до фактического окончания процесса
  загрузки.
- В случае прерывания загрузки уже загруженные файлы останутся на месте.

# ZipInputStream в Java

Для реализации описанного подхода очень удобным является язык программирования
Java по простой причине - весь стандартный ввод/вывод здесь стоит на двух
столпах: InputStream и OutputStream. InputStream представляет собой
поток произвольного набора байт и с которым вы можете делать только одну
операцию - читать из него. OutputStream это проивзольный поток байт в который
мы что-то пишем.

Главная прелесть InputStream состоит в том, что он универсален и не зависит от
источника получаемых данных. Мы можем получить его из любого набора байт, а не
только локального файла на компьютере. В том числе InputStream может быть
сформирован для удаленного файла на сервере, заполняясь по мере его загрузки.

Другой прелестью InputStream является то, что у него есть подклассы, которые
могут быть созданы из обычного InputStream и уточняют, с каким именно набором
данных мы имеем дело. К примеру FileInputStream представялет собой поток для
чтения из файлов на диске. Хорошая новость, что у Java есть встроенные средства
для работы с Zip архивами, и для нас главный интерес представляет класс
ZipInputStream, который предоствляет поток для чтения содержимого ZIP архивов.

У класса ZipInputStream есть метод getEntry(), который используется для
получения объекта ZipEntry, инкапсулирующего под собой файл или директорию
внутри архива. Если получение следующего файла внутри архива невозможно в
данный момент, то поток становится в состояние ожидания до тех пор пока будут
получены все байты из которых можно будет сформировать новый объект ZipEntry.

Итак, у нас есть все средства API для того чтобы реализовать распаковку на
лету.

# Реализация распаковки на лету на Java

Рассмотрим следующий код:

```java
    public static void unzipOnFly(URL url, Path unzipTo) {
        try (ZipInputStream zipInputStream = new ZipInputStream(url.openStream())) {
            int len;
            byte[] buffer = new byte[4096];
            for (ZipEntry entry = zipInputStream.getNextEntry(); entry != null; entry = zipInputStream.getNextEntry()) {
                System.out.println("Starting to decompress file: " + entry.getName());
                Path path = unzipTo.resolve(entry.getName());
                if (entry.isDirectory()) {
                    Files.createDirectory(path);
                } else {
                    File extractedfile = new File(entry.getName());
                    try (OutputStream outputStream = new FileOutputStream(extractedfile)) {
                        while ((len = zipInputStream.read(buffer)) != -1) {
                            outputStream.write(buffer, 0, len);
                        }
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
```

Данный метод принимает в качестве параметров ``URL url`` (прямую ссылку на
архив который нужно скачать) и путь ``Path unzipTo``, куда нужно распаковать
все файлы из архива. Из URL внутри метода открывается InputStream и начинается
процесс загрузки байтов Zip архива, которые попадют в поток. Так как мы знаем,
что по указанному URL у нас хранится именно архив, то мы можем сформировать из
изначального потока ZipInputStream, с помощью которого мы будем работать с
содержимым архива. Далее мы итерируемся по всем ``ZipEntry`` которые доступны
нам в данный момент при помощи ``zipInputStream.getNextEntry()`` до тех пор
пока мы не достигнем конца архива (``entry != null``). Внутри цикла получаем
имя текущего ZipEntry, разрешаем относительный путь, по которому мы должны
сохранить файл. Затем, если текущий ZipEntry является вложенной директорией в
архив, то мы просто создаем директорию с таким же именем по указанному пути.
Если это обычный файл, то мы создаем объект File и соответствующий ему
FileOutputStream - поток для записи содержимого файла из Zip архива. Мы читаем
из ZipInputStream при помощи метода ``InputStream.read()``, при этом метод
``read()`` возвращает количество считанных байт, которое мы сохраняем в
переменную ``len``, а сами байты попадают в переменную буфер, который мы
определили ранее ``byte[] readBuffer`` размером в 1 Кб (4096 байт). Мы передаем
этот буфер методу ``OutputStream.write()`` с указанием количества считанных
байт, чтобы записать файл из ZipInputStream на диск через FileOutputStream до
тех пор пока изначальный InputStream не вернет нам количество байт ``-1``
(конец потока).

Запустим следующий код, чтобы распаковать архив по ссылке на руководство ARU в
пустую директорию ``unzip``, действительно ли распавковка происходит как и
ожидалось на лету:

```java
    public static void main(String[] args) {
        try {
            Path unzipTo = Paths.get("/home/vasily/unzip");
            URL url = new URL("https://github.com/ventureoo/ARU/archive/refs/heads/main.zip");
            unzipOnFly(url, unzipTo);
        } catch (MalformedURLException e) {
            e.printStackTrace();
        }
    }
```

И результат:

```
Starting to decompress file: ARU-main/
Starting to decompress file: ARU-main/.gitignore
Starting to decompress file: ARU-main/.woodpecker.yml
Starting to decompress file: ARU-main/LICENSE
Starting to decompress file: ARU-main/README.md
Starting to decompress file: ARU-main/README.ru.md
Starting to decompress file: ARU-main/archive/
Starting to decompress file: ARU-main/archive/ARU/
Starting to decompress file: ARU-main/archive/ARU/ARU-2021.11.21-google-docs.docx
Starting to decompress file: ARU-main/archive/ARU/ARU-2021.11.21-google-docs.html
Starting to decompress file: ARU-main/archive/ARU/ARU-2021.11.21-google-docs.odt
Starting to decompress file: ARU-main/archive/ARU/ARU-2021.11.21-google-docs.pdf
Starting to decompress file: ARU-main/archive/ARU/images/
...
Starting to decompress file: ARU-main/docs/source/preface.html
Starting to decompress file: ARU-main/docs/source/preface.rst
Starting to decompress file: ARU-main/docs/source/useful-programs.html
Starting to decompress file: ARU-main/docs/source/useful-programs.rst
```

Мгновенная распаковка на лету архива размером 15 Мегабайт! Магия вне Хогвартса.

Но у представленного кода есть некоторые изъяны, которые можно исправить. Прежде
всего запись файлов лучше всего производить не через ``FileOutputStream``, а
``BufferedOutputStream``, но и в целом использование переменной ``byte[]
buffer`` не является оптимальным, так как её размер фиксирован и создает
промежуточные значения в памяти. При записи больших файлов из архива это может
привести к замедлению общей производительности распаковки, так как перед тем
как перейти к обработке следующего файла архива программа будет дожидаться
окончания записи. Поэтому лучше всего использовать класс ``FileChannel`` из
пакета Java NIO. Мы не будем сейчас долго останавливаться на специфике работы
New I/O в Java, так как это выходит за рамки данной статьи, всё что нужно знать
про FileChannel и его метод ``transferFrom`` для записи - это то, что они гораздо
более производительны чем запись с использованием буфера, так как использование
данного метода позволяет записывать получаемые данные напрямую в кэш файловой
системы, не создавая при этом промежуточных копий.

Ещё одной проблемой является то, что мы выполняем загрузку файла при помощи
``URL.openStream()``. Данный метод не даёт полного контроля над процессом
загрузки не позволяет нормально обрабатывать возникаемые ошибки при установке
соединения и передаче данных. Вместо этого разумеется следует использовать
библиотеку HTTP. Библиотека может быть на ваш вкус, главное условие - это
возвращение именно ``InputStream`` в качестве тела получаемого ответа от
сервера.

Перепишем используемый метод с использованием новой встроенной библиотеки
``java.net.http``, появившейся в Java 11:

```java
    public static void unzipOnFly(URI url, Path unzipTo) {
        try {
            HttpClient client = HttpClient.newBuilder().build();
            HttpRequest request = HttpRequest.newBuilder().uri(url).build();
            HttpResponse<InputStream> response = client.send(request, HttpResponse.BodyHandlers.ofInputStream());

            if (response.statusCode() != 200) {
                System.out.println("HTTP error occurred! Status code: " + response.statusCode());
                return;
            }

            try (ZipInputStream zipInputStream = new ZipInputStream(response.body())) {
                for (ZipEntry entry = zipInputStream.getNextEntry(); entry != null; entry = zipInputStream.getNextEntry()) {
                    System.out.println("Starting to decompress file: " + entry.getName());
                    Path toPath = unzipTo.resolve(entry.getName());
                    if (entry.isDirectory()) {
                        Files.createDirectory(toPath);
                    } else try (FileChannel fileChannel = FileChannel.open(toPath, CREATE, StandardOpenOption.WRITE)) {
                        fileChannel.transferFrom(Channels.newChannel(zipInputStream), 0, Long.MAX_VALUE);
                    }
                }
            }
        } catch (IOException e) {
            e.printStackTrace();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
```

Здесь мы сначала инициируем HTTP клиент для отправки запросов, формируем сам
запрос и отправляем его с указанием обработчика тела запроса как
``BodyHandlers.ofInputStream()``, который должен вернуть нам в качестве тела
объект ``InputStream``, а не строку. Появляется возможность обработки ошибок на
уровне HTTP. Код записи файлов заметно упростился при использовании
``FileChannel``, теперь у нас нет лишних переменных ``len`` и ``byte[]
buffer``, а сама запись стала быстрее.

Напишем ещё один пример, но уже с использованием асинхронной отправки HTTP
запросов (метода sendAsync):

```java
public class AsyncExample {
    private final static ExecutorService service = Executors.newFixedThreadPool(4);
    private final static HttpClient client = HttpClient.newBuilder()
            .followRedirects(HttpClient.Redirect.ALWAYS)
            .executor(service)
            .build();

    public static void main(String[] args) {
        Path unzipTo = Paths.get("/home/vasily/unzip");
        URI link = URI.create("https://github.com/ventureoo/ARU/archive/refs/heads/main.zip");
        try {
            unzipOnFly(link, unzipTo);
            service.awaitTermination(1L, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void unzipOnFly(URI url, Path unzipTo) throws InterruptedException {
        HttpRequest request = HttpRequest.newBuilder().uri(url).build();
        client.sendAsync(request, HttpResponse.BodyHandlers.ofInputStream())
                .thenApply(response -> response.body())
                .thenAccept(stream -> {
                    try (ZipInputStream zipInputStream = new ZipInputStream(stream)) {
                        for (ZipEntry entry = zipInputStream.getNextEntry(); entry != null; entry = zipInputStream.getNextEntry()) {
                            System.out.println("Starting to decompress file: " + entry.getName());
                            Path toPath = unzipTo.resolve(entry.getName());
                            if (entry.isDirectory()) {
                                Files.createDirectory(toPath);
                            } else try (FileChannel fileChannel = FileChannel.open(toPath, CREATE, StandardOpenOption.WRITE)) {
                                fileChannel.transferFrom(Channels.newChannel(zipInputStream), 0, Long.MAX_VALUE);
                            }
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
    }
}
```

Здесь мы имеем все то же самое, но применяем уже средства продвинутой
многопоточности в Java как ``CompletableFuture`` и ``ExecutorService``,
добавляя каллбэк в ``thenApply`` передавая InputStream вместо HttpResponse и
каллбэк ``thenAccept`` для получения этого InputStream и распаковки архива. Сам
метод ``sendAsync`` возвращает нам по сути промис (``Future<Void>``). Хотя в
примере по прежнему создается только один запрос, таким образом можно удобно
создавать большое количество запросов на одновременную загрузку архивов,
ограничивая при этом количество одновременно загружаемых архивов через указания
собственного ``ExecutorService`` HTTP клиенту.

# Использование библиотеки Zip4J

К сожалению встроенное API для работы с ZIP архивами в Java несколько
ограниченно. Например, вы не сможете с его помощью распаковать архив если
он будет защищен паролем или методами шифрования AES. На помощь в этом случае
приходит сторонняя библиотека для работы с архивами - Zip4J. Она поддерживает
всё вышеперечисленное и даже больше, позволяя нам например отслеживать процесс
расжатия архива через ProgressBar.

По сути главными различиями между Zip4J и встроенным API в Java является то,
что для получения InputStream мы используем класс ZipInputStream из пакета
``net.lingala.zip4j.io.inputstream``, а ZipEntry заменяется на класс
``net.lingala.zip4j.model.LocalFileHeader``. Для работы с зашифрованными
архивами мы должны передать дополнительный параметр ``char[] password``.

Вот переделанный метод загрузки с использованием библиотеки Zip4J:

```java
public class AsyncExample {
    private final static ExecutorService service = Executors.newFixedThreadPool(4);
    private final static HttpClient client = HttpClient.newBuilder()
            .followRedirects(HttpClient.Redirect.ALWAYS)
            .executor(service)
            .build();

    public static void main(String[] args) {
        Path unzipTo = Paths.get("/home/vasily/unzip");
        URI link = URI.create("https://github.com/ventureoo/ARU/archive/refs/heads/main.zip");
        try {
            unzipOnFly(link, unzipTo);
            service.awaitTermination(1L, TimeUnit.MINUTES);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    public static void unzipOnFly(URI url, Path unzipTo) throws InterruptedException {
        HttpRequest request = HttpRequest.newBuilder().uri(url).build();
        client.sendAsync(request, HttpResponse.BodyHandlers.ofInputStream())
                .thenApply(response -> response.body())
                .thenAccept(stream -> {
                    try (ZipInputStream zipInputStream = new ZipInputStream(stream)) {
                        for (LocalFileHeader entry = zipInputStream.getNextEntry(); entry != null; entry = zipInputStream.getNextEntry()) {
                            System.out.println("Starting to decompress file: " + entry.getFileName());
                            Path toPath = unzipTo.resolve(entry.getFileName());
                            if (entry.isDirectory()) {
                                Files.createDirectory(toPath);
                            } else try (FileChannel fileChannel = FileChannel.open(toPath, CREATE, StandardOpenOption.WRITE)) {
                                fileChannel.transferFrom(Channels.newChannel(zipInputStream), 0, Long.MAX_VALUE);
                            }
                        }
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                });
    }
}
```

На это всё, надеюсь эта статья была кому-то полезна.
