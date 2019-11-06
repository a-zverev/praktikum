*версия 2, исправленная*

## Работа на удаленном сервере
Управление сервером по протоколу ssh. Мы можем получить доступ как к удаленному терминалу, так и к файловой системе удаленного сервера.
Для работы с удаленным сервером нужно знать несколько параметров:

* адрес сервера: 178.236.129.66
* порт для подключения: 9911
* логин и пароль: 

#### Linux
Утилита для удаленного управления терминалом ssh - уже установлена.  Для монтирования разделов сервера к вашей файловой системе понадобится sshfs. Инструкции по установке [здесь](https://habr.com/en/post/52310/) 


* Подключение к удаленному терминалу:

`ssh perstuds@178.236.129.66 -p 9911`

После подключения имя пользователя и имя вашего локального компьютера, указанное у вас в терминале, изменится на имя аккаунта на сервере. В нашем случае это `perstuds@arriam-lab2-cpu`

Для работы нам понадобится передавать и принимать файлы с сервера. Удобнее всего это сделать при помощи монтирования файловой системы сервера к файловой системе локальной машины. Создайте на своей машине папку, которая будет точкой монтирования - в моем примере это папка `Praktikum` в домашней папке текущего пользователя.(Обратите внимание, что монтируем разделы мы командой из терминала локальной машины, не сервера)

* Монтирование разделов сервера: 

`sshfs perstuds@178.236.129.66:/home/perstuds/ /home/alister_ari/Praktikum -p 9911`


#### Windows
Для удаленной работы нужно скачать и установить ssh-клиент. Например, Bitwise SSH Client - его можно скачать [здесь](https://www.bitvise.com/ssh-client-download). В соответствующих полях заполните адрес сервера, порт, имя пользователя и пароль. После подключения вам должен быть доступен удаленный терминал и окно обмена файлами.

### Правила безопасности

Поскольку мы все работаем от имени одного пользователя, у нас единое рабочее пространство. Чтобы не мешать друг другу, обязательное требование - каждый должен создать свою папку и работать только в ней.

* Ваша рабочая директория должна выглядеть как `/home/perstuds/your_name/`

## Менеджер сессий tmux
Чтобы не потерять данные после закрытия терминала, создайте свою сессию в менеджере сессий. Задачи, запущенные в tmux, остаются активными после обрыва соединения.

* Создать именованную сессию: `tmux new-session -s your_name`
* Посмотреть список сессий: `tmux ls`
* Подключиться к текущей сессии `tmux attach -t your_name`

## Утилиты и исходные данные

Для работы нам понадобятся утилиты trimmomatic, fastq-join и vsearch. Они установлены в окружении bioinf. Активируйте окружение:

    conda activate bioinf

B проверьте, что все нужные программы вам доступны, вызвав справки:

    trimmomatic -h
    fastq-join -h
    vsearch -h

Вам также потребуются следующие данные:

* файлы прочтений, прямые и обратные (находятся в `/home/perstuds/source/seqs`)
* файл карты (map, находится в `/home/perstuds/source/map.txt`). Это файл таблица с разделителями табуляцией, содержащий метаданные о наших образцах. 
* референсная база данных. Мы используем базу [SILVA 123](https://www.arb-silva.de/documentation/release-123/), предоставляющую последовательности 16s региона, таксономическую принадлежность этих последовательностей, а также файл дерева - информацию о сходстве последовательностей.


Обратите внимание, для наших дальнейших вычислений важно форматирование карты:
* в первой графе должны быть названия образцов
* во второй - RawFilenamePrefix, или префиксы файлов сиквенсов (имена без _L001_R[12]_001)
* Графа Filename, названия файлов в них должны соответствовать именам образцов, расширения - .fna
* Последняя графа должна называться Description.


Скопируйте к себе в рабочую директорию файлы сиквенсов и карту.

    mkdir raw_data
    cp /home/perstuds/source/seqs/*.* /home/perstuds/zverev/raw_data
    cp /home/perstuds/source/map.txt /home/perstuds/zverev/
    

## Тримминг и объединение последовательностей

Для удаления ошибочно прочтитанных последовательностей и объединения прямых и обратных ридов мы используем [trimmomatic](http://usadellab.org/cms/?page=trimmomatic). Для объединения последователностей - [fastq-join](https://github.com/brwnj/fastq-join). Чтобы не считать все по одному файлу, воспользуемся циклом bash.

Для начала создадим файл с названиями наших образцов, используя файл карты `map.txt`:
```bash
    awk '(NR>1) {print$2}' map.txt > samples.txt
```

Тримминг и объединение последовательностей (не забудьте активировать окружение bioinf):
```bash
    mkdir trimmed
    mkdir tmp_trimmed
    for sample in $(cat samples.txt); do
        echo ${sample};
        dirname="tmp_trimmed/${sample}";
        forward="raw_data/${sample}_L001_R1_001.fastq.gz";
        reverse="raw_data/${sample}_L001_R2_001.fastq.gz";
        mkdir ${dirname};
        trimmomatic PE -phred33 \
                ${forward} \
                ${reverse} \
                ${dirname}/trimed_paired_forward.fastq.gz \
                ${dirname}/trimed_upaired_forward.fastq.gz \
                ${dirname}/trimed_paired_reverse.fastq.gz \
                ${dirname}/trimed_unpaired_reverse.fastq.gz \
                SLIDINGWINDOW:4:12 MINLEN:180;
        fastq-join \
                ${dirname}/trimed_paired_forward.fastq.gz \
                ${dirname}/trimed_paired_reverse.fastq.gz \
                -o ${dirname}/trimmed_%.fastq.gz;
        cp ${dirname}/trimmed_join.fastq.gz \
        trimmed/${sample}.fastq.gz;
        done
    rm tmp_trimmed -r
```
Теперь наши файлы, почищенные и объединенные, доступны в папке `trimmed`.


## Удаление химер

Мы будем использовать [vsearch](https://github.com/torognes/vsearch) - многофункциональный инструмент для работы с fasta-файлами. В утилите доступно два режима поиска химерных последовательностей - поиск химер de novo и по базе данных. Поскольку у нас есть достаточно полная база для нашего участка, мы будем использовать именно последний вариант. Это самый ресурсозатратный шаг обработки может занимать до нескольких часов.
```bash
    mkdir dechimered
    mkdir tmp_dechimered
    databasefna='/home/perstuds/tax_n_refs/silva_132_97_16S.fna'
    for sample in $(cat samples.txt); do
        echo ${sample};
        dirname="tmp_dechimered/${sample}";
        mkdir ${dirname}
        cp trimmed/${sample}.fastq.gz ${dirname}/${sample}.fastq.gz
        gzip -d ${dirname}/${sample}.fastq.gz;
        sed -n '1~4s/^@/>/p;2~4p' \
                    ${dirname}/${sample}.fastq > ${dirname}/${sample}.fna
        vsearch \
            --uchime_ref ${dirname}/${sample}.fna\
            --nonchimeras ${dirname}/dechim_${sample}.fna\
            --db ${databasefna} --threads 6;
        mv ${dirname}/dechim_${sample}.fna \
            dechimered/${sample}.fna;
        done
    rm -r tmp_dechimered
```
По результатам работы файлы доступны в директории `dechimered`

## ОТЕ-пикинг

Все последующие шаги мы будем делать при помощи скриптов пакета [QIIME](http://qiime.org/). Этот пакет установлен в отдельном окружении, так что поменяйте окружение на `qiime1`

    conda activate qiime1

Для того, чтобы перейти непосредственно к ОТЕ-пикингу, необходимо создать единый fasta-файл для всех образцов. Для того, чтобы впоследствии знать, какая последовательность принадлежит какому из образцов, нужно добавить в заголовки последовательностей информацию, из какой пробы они взяты. Делается это при помощи скрипта [add_qiime_labels.py](http://qiime.org/scripts/add_qiime_labels.html):

    add_qiime_labels.py -i dechimered -m map.txt -c Filename -o combined_fasta

Теперь, если вы посмотрите заголовки последовательностей в `/combined_fasta/combined_seqs.fna`, вы убедитесь, что эта информация там есть.

Непосредственно ОТЕ-пикинг мы будем делать по методу closed-reference, поскольку в нашем распоряжении есть хорошая база данных по изучаемой последовательности. Воспользуемся скриптом [pick_closed_reference_otus.py](http://qiime.org/scripts/pick_closed_reference_otus.html):

    pick_closed_reference_otus.py -i /home/perstuds/zverev/combined_fasta/combined_seqs.fna -r /home/perstuds/tax_n_refs/silva_132_97_16S.fna -o otus -t home/perstuds/tax_n_refs/taxonomy_7_levels.txt

По завершении ОТЕ-пикинга давайте выведем немного статистики по нашим образцам, и конвертируем в текстовый вид (информация о пакете [biom-format](http://biom-format.org/index.html)):

    biom summarize-table -i otus/otu_table.biom
    biom convert -i otus/otu_table.biom -o otus/otu_table.txt --to-tsv --header-key taxonomy
    

## Фильтры.

QIIME предоставляет большой набор инструментов для анализа и корректировки нашей выборки. Вот лишь несколько из вариантов:

* удалить образцы, общее число прочтений в которых ниже порога
* удалить ОТЕ, принадлежащие к определенному таксону
* удалить ОТЕ,встречающиеся слишком редко
* объединить повторности
* нормировать выборку

Полный список скриптов с кратким описанием доступен в разделе [Scripts](http://qiime.org/scripts/) Воспользуемся некоторыми из них. В частности, мы явно хотим исключить из анализа прочтения, принадлежащие последовательностям митохондрий и хлоропластов (скрипт [filter_taxa_from_otu_table.py](http://qiime.org/scripts/filter_taxa_from_otu_table.html)):

    filter_taxa_from_otu_table.py -i otus/otu_table.biom -o otus/f_otu_table.biom -n D_3__Chloroplast,D_4__Mitochondria

А также сортировать образцы по именам ([sort_otu_table.py](http://qiime.org/scripts/sort_otu_table.html)):

    sort_otu_table.py -i otus/f_otu_table.biom -o otus/sf_otu_table.biom

После этого давайте посмотрим барграфы представленности наших таксонов. Это удобная форма представления данных, позволяет получить представление о структуре сообщества (скрипт [summarize_taxa_through_plots.py](http://qiime.org/scripts/summarize_taxa_through_plots.html))

    summarize_taxa_through_plots.py -o taxa_summary -i otus/sf_otu_table.biom -m map.txt

В директории `/taxa_summary` теперь есть барграфы и барплоты. Посмотрев на них, можно принять решение, стоит ли убрать некоторые образцы-выбросы, хорошо ли повторности повторяют друг друга, и какие таксоны являются превалирующими.

Следующий шаг, который нужно сделать, это нормировка. Стоит ли использовать нормировку вообще, и если да, то какую именно - вопрос дискуссионный, в статьях можно встретить самые разные варианты. В демонстрационных целях мы используем самый простой вариант нормировки, а именно разрежение (rarefaction). Нормировать мы будем на наименьшее число последовательностей среди образцов (скрипт [single_rarfaction.py](http://qiime.org/scripts/single_rarefaction.html).


    single_rarefaction.py -i otus/sf_otu_table.biom -o otus/nsf_otu_table.biom.biom -d 12856

Также для некоторых анализов можно использовать информацию о таксономческой близости некоторых ОТЕ. Эта информация находится в файле дерева, которое есть в составе нашей базы референсов. Поскольку в наших образцах присутствуют не все ОТЕ, представленные в базе, необходимо обрезать дерево (скрипт [filter_tree.py](http://qiime.org/scripts/filter_tree.html)), удалив оттуда листья, которые не представлены в нашей таблице. Для этого нам понадобится список всех ОТЕ, включенных в таблицу. Проще всего получить его конвертированием таблицы в формат .txt и чтением первого столбца
```bash
    biom convert -i otus/nsf_otu_table.biom -o otus/nsf_otu_table.txt --to-tsv
    awk '(NR>3) {print$1}' otus/nsf_otu_table.txt > otus/otu_list.txt
    filter_tree.py -i /home/home/perstuds/tax_n_refs/97_otus.tre -t otus/otu_list.txt -o otus/filtered.tre
```

## Альфа- и бета-разнообразие

Теперь все готово для того, чтобы посчитать наконец индексы разнообразия. Используем скрипты [alpha_diversity.py](http://qiime.org/scripts/alpha_diversity.html) и [beta_diversity_through_plots.py](http://qiime.org/scripts/beta_diversity_through_plots.html). Помимо стандартных, вы можете использовать разные метрики.

    mkdir alpha_div
    alpha_diversity.py -i otus/nsf_otu_table.biom -m PD_whole_tree,chao1,observed_otus,shannon -o alpha_div/indexes.txt -t otus/filtered.tre
    beta_diversity_through_plots.py -i otus/nsf_otu_table.biom -o beta_div -t otus/filtered.tre -m map.txt



## Современные инструменты анализа 16s rDNA сообществ

Мы рассмотрели QIIME - первый наиболее полный и функцональный пакет для обработки метагеномных данных. На сегодняшний день существуют и более современные версии. Они сложнее в освоении, но более эффективны. В первую очередь это [QIIME2](https://qiime2.org/), работающий непосредственно из оболочки Bash, и [dada2](https://benjjneb.github.io/dada2/tutorial.html), библиотека для R.


