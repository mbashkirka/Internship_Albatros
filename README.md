# Стажировка в компании "Альбатрос"
Стажировка направлена на разработку программы, которая поможет компании минимизировать потери из-за недопоставок и нехватки товара. Основная бизнес-задача заключается в анализе текущих запасов и прогнозировании потребностей, чтобы избежать дефицита. Итоговая программа будет автоматически рассчитывать необходимое количество товара, основываясь на исторических данных и текущих трендах, что позволит компании оптимизировать свои закупки и улучшить уровень обслуживания клиентов.
## Личное участие
За время стажировки я успел поработать в двух этапах, в первом и третьем. Первый этап (сбор данных) оказался достаточно простым и быстрым из-за чего так и получилось, что я оказался в команде разработки алгоритма, 
## Разработка
### Первый этап - Сбор данных
Задача стояла такая: извлечь все необходимые данные из сервисов Ozon Seller и Google Sheets через API. Эти данные важны для прогнозирования и расчета нужного количества товара. Привожу два примера, 1 пример - функция, выгружающая информацию об остатках товара на складе Ozon, 2 пример - функция, выгружаюшая данные о товаре и его маржинальности из таблицы Google Sheets:
```
def stock_on_warehouses(limit, offset, warehouse_type):
    url = "https://api-seller.ozon.ru/v2/analytics/stock_on_warehouses"
    payload = {
        "limit": limit,
        "offset": offset,
        "warehouse_type": warehouse_type
    }
    headers = {
        "Content-Type": "application/json",
        "Client-Id": client_id,
        "Api-Key": oz_token
    }
    response = requests.post(url, json=payload, headers=headers)
    print("Ответ сервера при запросе об остатках на складе Озона:", response.status_code)
    if response.status_code == HTTPStatus.OK:
        request_data = response.json()
        df_json = pd.read_json(json.dumps(request_data))
        rows = df_json['result'].values[0]
        df = pd.DataFrame([{
            'sku': item['sku'],
            'item_code': item['item_code'],
            'item_name': item['item_name'],
            'free_to_sell_amount': item['free_to_sell_amount'],
            'promised_amount': item['promised_amount'],
            'warehouse_name': item['warehouse_name']
        } for item in rows])
        output_file = os.path.join(output_directory, "stock_on_warehouses.csv")
        df.to_csv(output_file, index=False, encoding='utf-8-sig')
        return df
    elif response.status_code == HTTPStatus.TOO_MANY_REQUESTS:
        print('Превышен лимит на запросы в 1 мин.')
    elif response.status_code == HTTPStatus.BAD_REQUEST:
        print('Неверный параметр.')
    elif response.status_code == HTTPStatus.FORBIDDEN:
        print('Доступ запрещён.')
    elif response.status_code == HTTPStatus.NOT_FOUND:
        print('Ответ не найден.')
    elif response.status_code == HTTPStatus.CONFLICT:
        print('Конфликт запроса.')
    elif response.status_code == HTTPStatus.INTERNAL_SERVER_ERROR:
        print('Внутренняя ошибка сервера.')
    else:
        print(f"Что-то пошло не так! Ошибка отличная от 400, 403, 404, 409, 429, 500.")
```
```
def marginality():
    sh = gc.open("24' ОБЩИЕ ПРОДАЖИ")
    data = sh.sheet1.get_all_values()
    non_empty_rows = sum(1 for row in data if row[0])
    filtered_data = [[data[i][0], data[i][13]] for i in range(1, non_empty_rows)]
    output_file = os.path.join(output_directory, 'marginality.csv')
    with open(output_file, 'w', newline='', encoding='utf-8-sig') as file:
        writer = csv.writer(file)
        writer.writerows(filtered_data)
    return filtered_data
```
### Второй этап - Аналитика и прогнозирование
Задача заключалась в том, чтобы на основе данных, полученных от команды сбора данных, подобрать алгоритм машинного обучения или статистики, который наиболее точно предскажет результаты. В итоге был выбран алгоритм FB Prophet, который сейчас используется для всех прогнозов.
### Третий этап - Алгоритм и разработка
Третий и заключительный этап включал разработку алгоритма, который на основе всех собранных и прогнозируемых данных формирует отчет. В этом отчете определяется момент, когда текущий запас товаров исчерпается, а также рассчитывается потеря прибыли на данный момент. Такой отчет предоставляет руководителю компании четкое представление о том, какие товары и в каком количестве необходимо заказать, чтобы избежать недопоставок и обеспечить наличие товаров для покупателей.
```
def report(date_start, metrics, dimensions, filters, sort, limit, offset, warehouse_type, list_status):
    df_report = pd.DataFrame(columns=['offer_id', 'stock_Albatros', 'stock_Ozon', 'goods_in_purchases', 'stock_end', 'fastest_purchase', 'lost_profits', 'required_order'])

    df_sales_analytics = sales_analytics(date_start, metrics, dimensions, filters, sort, limit, offset)

    assortment_data, offer_id_uniq = assortment()

    sklad_data = sklad()
    sklad_dict = {row[0]: row[1] for row in sklad_data}

    purchases_data = purchases(list_status)
    purchases_dict = {}
    purchases_dict_help = {}
    for row in purchases_data:
        date_on = datetime.datetime.strptime(row[1], "%d/%m/%Y")
        offer_id = row[2]
        amount = int(row[3].replace(',', ''))
        if offer_id not in purchases_dict:
            purchases_dict[offer_id] = []
            purchases_dict_help[offer_id] = []
        purchases_dict[offer_id].append((date_on, amount))
        purchases_dict_help[offer_id].append(amount)

    assortment_dict = {}
    for row in assortment_data:
        offer_id = row[0]
        production_time = row[1]
        purchase_type = row[2]
        if offer_id not in assortment_dict:
            assortment_dict[offer_id] = []
        assortment_dict[offer_id].append((production_time, purchase_type))

    df_forecasting = pd.read_csv('~/forecasting.csv')

    df_stock_on_ozon = calculate_stock_summary_ozon(stock_on_warehouses(limit, offset, warehouse_type))

    for offer_id in offer_id_uniq:
        row_stock_on_ozon = df_stock_on_ozon[df_stock_on_ozon['item_code'] == offer_id]
        row_forecasting = df_forecasting[df_forecasting['offer_id'] == int(offer_id)].iloc[0, 1:54].values
        row_sales_analytics = df_sales_analytics[df_sales_analytics['offer_id'] == offer_id]

        if offer_id in sklad_dict:
            stock_Albatros = sklad_dict.get(offer_id)
        else:
            stock_Albatros = 0

        if not row_stock_on_ozon.empty:
            stock_Ozon = row_stock_on_ozon['total_amount'].values[0]
        else:
            stock_Ozon = 0

        if offer_id in purchases_dict_help:
            goods_in_purchases = sum(purchases_dict_help[offer_id])
        else:
            goods_in_purchases = 0

        result_stock = int(stock_Ozon) + int(stock_Albatros)
        weeks_remaining = 0
        i = current_week()
        if offer_id in purchases_dict:
            while result_stock > 0 or len(purchases_dict[offer_id]) > 0:
                for date_on, amount in purchases_dict[offer_id]:
                    if date_on.isocalendar()[1] == i:
                        result_stock += amount
                        purchases_dict[offer_id].remove((date_on, amount))
                if result_stock - row_forecasting[i - 1] >= 0:
                    weeks_remaining += 1
                    result_stock -= row_forecasting[i - 1]
                else:
                    if len(purchases_dict[offer_id]) > 0:
                        weeks_remaining += 1
                    else:
                        break
                i = (i + 1) % 53
                if i == 0:
                    i = 1
        else:
            while result_stock > 0:
                if result_stock - row_forecasting[i - 1] < 0:
                    break
                else:
                    weeks_remaining += 1
                    result_stock -= row_forecasting[i - 1]
                i = (i + 1) % 53
                if i == 0:
                    i = 1
        stock_end, stock_real_end = get_week_dates(weeks_remaining)

        prices = row_sales_analytics['price'].values
        valid_prices = [price for price in prices if price != 0]
        if valid_prices:
            result_price = round(sum(valid_prices) / len(valid_prices))
        else:
            result_price = 0

        fastest_purchase = add_days_to_date_google(datetime.datetime.now().strftime("%d/%m/%Y"), 35)
        start_date_str, end_date_str = stock_real_end.split(' - ')
        end_date = datetime.datetime.strptime(end_date_str, '%d/%m/%Y')
        fastest_purchase_date = datetime.datetime.strptime(fastest_purchase, '%d/%m/%Y')
        stock_promises = 0
        if end_date <= fastest_purchase_date:
            weeks_promises = math.ceil(date_difference(fastest_purchase_date, end_date) / 7)
            for i in range(current_week() + weeks_remaining + 1, current_week() + weeks_remaining + 1 + weeks_promises):
                j = i % 53
                if j == 0:
                    j = 1
                stock_promises += row_forecasting[j-1]
            lost_profits = -(stock_promises * result_price)
        else:
            lost_profits = 0

        if weeks_remaining >= 24:
            required_order = 0
        else:
            remaining_weeks = 24 - weeks_remaining
            forecast_for_remaining_weeks = 0
            for week in range(remaining_weeks):
                forecast_week_index = (i + week) % len(row_forecasting)
                forecast_for_remaining_weeks += row_forecasting[forecast_week_index]
            required_order = max(forecast_for_remaining_weeks - result_stock, 0)

        df_report = df_report._append({
            'offer_id': offer_id,
            'stock_Albatros': stock_Albatros,
            'stock_Ozon': stock_Ozon,
            'goods_in_purchases': goods_in_purchases,
            'stock_end': stock_end,
            'fastest_purchase': fastest_purchase,
            'lost_profits': lost_profits,
            'required_order': required_order
        }, ignore_index=True)
    df_report = df_report.sort_values(by='lost_profits', ascending=True)
    return df_report
```