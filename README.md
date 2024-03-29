# Соревнование [NLP in practice](https://ods.ai/competitions/nlp-receipts) от Альфа Банк, проходившие на платформе ods.ai июне-августе 2023 года

## Задача - структуризации чеков ОФД
Данные чеков ОФД содержат детальную информацию о тратах клиентов. Они помогают улучшать качество моделей кредитного скоринга и склонности к банковским продуктам, а также улучшать пользовательский опыт за счет структуризации трат клиентов в мобильном приложении. Однако работа с этим источником затрудняется его неструктурированностью: вся информация о купленном товаре лежит в одной строке произвольного формата.

В предположении, что каждая чековая позиция описывает какой-либо товар, наименование этого товара, а также его бренд, являются главной информацией, которую можно извлечь из чека. По итогу задача структуризации этих данных ограничивается выделением и нормализацией брендов и товаров.

## Что я делал

Я проводил эксперименты по подбору модели и гиперапараметров для нее, а также пробовал различные способы предобработки данных. Задачу я свел к NER, то есть выделению именнованных сущностей. По сути, классификации слов в предложении на good, brand и все остальное. 

Модели, которые пробовал: **BERT, RNN, GRU** (в т.ч. bidirectional). Эсперементировал с токенизатором: **fasttext и BertTokenizer**. В итоге,  **лучший score F1 = 0,727** был достигнут на сочетании BERT и BertTokenizer, с использованием [предобученной русскоязычной модели](https://huggingface.co/cointegrated/rubert-tiny2) с HuggingFace, которая называется rubert-tiny. Посмотреть решение можно в **NER_with_BERT.ipynb**.

После серии экспериментов, была выбрана следующая **предобработка строк**: 
```python
def normalize_str_test(x_str):
    # убираем всё, кроме букв и цифр
    current_str = re.sub(r"[^a-zA-ZА-Яа-я0-9]", " ", x_str)
    # заменяем все числа на 1
    current_str = re.sub('[0-9]+', '1', current_str)
    # разделяем camelCase
    current_str = re.sub(r'([^A-Z])([A-Z])', r'\1 \2', current_str).lower().split()
    for n, word in enumerate(current_str):
        # убираем слова длинной в 1 символ
        if len(word) == 1:
            current_str.remove(word)
            continue
        is_mixed = is_rueng(word)
        if is_mixed:
            if is_mixed == 'rus':
                abc1 = 'ё' + 'etyopahkxcbmin' + string.punctuation
                abc2 = 'е' + 'етуоранкхсвмин' + ' ' * len(string.punctuation)
            else:
                abc1 = 'етуоранкхсвмли' + string.punctuation
                abc2 = 'etyopahkxcbmli' + ' ' * len(string.punctuation)
            trans_dict = str.maketrans(dict(zip(abc1, abc2)))
            word = word.translate(trans_dict)
            current_str[n] = word
    return ' '.join(current_str)
```

## Чему я научился 
- работать с BERT и RNN аритектурами
- проводить большое количество экспериментов
- обучать тяжелые модели
- решать задачи по NER
- делать BIO-tagging
- полностью погружаться в данные
- искать инсайты 
## Данные
Участникам соревнования предоставляются два датасета с чековыми позициями, размеченный и неразмеченный:

В размеченном датасете для каждой чековой позиции указаны нормализованные бренды и товары входящие в нее в исходном виде.
В неразмеченном датасете даны только сами чековые позиции.
Подробное описание файлов и полей датасета соревнования можно найти во вкладке Данные.

### Пример данных
| name | good | brand |
|------|------|-------|
| Вода "Саирме" с/г 500мл | вода | sairme |
| Моя Семя . 0.175л и ассортим | | моя семья |
| Рулет бисквитн.Яшкино клубничный со слив | рулет | яшкино |
| 460075794371 Почвогрунт Цветочное счастье Фас... | почвогрунт | фаско |
| Семечки жар 80г Крутой Окер с солью Гринвич | семечки | крутой окер |
		

## Метрика соревнования
Метрика соревнования - (F1_good + 2 * F1_brand) / 3, где F1_good- это метрика F1 на товарах, а F1_brand - это метрика F1 на брендах. Для подсчета F1 мы для считаем две метрики:

Precision - доля сущностей из предсказания модели, которые оказались верными (то есть присутствуют в ответе для соответствующих примеров)
Recall - доля сущностей в ответах, которые выделила модель
После чего метрика считается по знакомой формуле F1 = 2 / (1 / precision + 1 / recall)