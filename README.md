* Импорт БД:
	* mongoimport --db users ./users.json
* Результат импорта:
  * 2018-05-15T02:23:30.288+0300    no collection specified
  * 2018-05-15T02:23:30.295+0300    using filename 'users' as collection
  * 2018-05-15T02:23:30.432+0300    connected to: localhost
  * 2018-05-15T02:23:30.836+0300    imported 844 documents
  
# 0 - Накатить бэкап базы
  `создал папку backups`
 * Запрос:
    * mongodump --db users --out ./backups/
 * Ответ:
    * 2018-05-15T02:37:52.834+0300    writing users.users to
    * 2018-05-15T02:37:52.851+0300    done dumping users.users (844 documents)

# 1 - Найти средний возраст людей в системе
  * Запрос:
    * db.users.aggregate([{$group: {_id: "Averaga age", age: {$avg: "$age"}}}]);
  
  * Ответ:
     * { "_id" : "Averaga age", "age" : 30.38862559241706 }

# 2 - Найти средний возраст в штате Аляска
  * Запрос:
    * db.users.aggregate([{$match: {address: {$regex: /(Alaska)/}}}, {$group: {_id: "avg_age", avg_alaska: {$avg: "$age"}}}]);
  
  * Ответ:
    * { "_id" : "avg_age", "avg_alaska" : 31.5 }
    
# 3 - Начиная от Math.ceil(avg + avg_alaska) (порядковый номер документа в БД ) найти первого человека с другом по имени Деннис
  #### Подготовка к запросу: 
  * Создал курсор по среднему возрасту во всей системе
    * var avg = db.users.aggregate([{$group: {_id: "Averaga age", age: {$avg: "$age"}}}]);null;
    * var avg_common = avg.hasNext() ? avg.next() : null;
    * avg_common["age"]; --> 30.3886
  * Создал курсор по среднему возрасту в Аляске
    * var avg_alaska = db.users.aggregate([{$match: {address: {$regex: /(Alaska)/}}}, {$group: {_id:"avg_age",  avg_alaska: {$avg: "$age"}}}]);null;
    * var avg_alaska_age = avg_alaska.hasNext() ? avg_alaska.next() : null;
    * avg_alaska_age["avg_alaska"];  --> 31.5
  ###### Math.ceil(avg_common["age"] + avg_alaska_age["avg_alaska"]) = 62;     
  `Запрос производился с учетом английского стандарта записи полного имени - Имя - Фамилия`  
  
  * Запрос через индекс:
    * db.users.find({$and: [{friends: {$elemMatch: {name: {$regex: /(Dennis )/}}}}, {index: {$gt: Math.ceil(avg_common["age"]+avg_alaska_age["avg_alaska"])}}]}, {address: 1,  name: 1, friends:
  1}).limit(1).pretty();
  
  * Ответ:
    * {
        "_id" : ObjectId("5adf3c1544abaca147cdd539"),
        "name" : "Keller Nixon",
        "address" : "591 Jamison Lane, Idamay, Minnesota, 3128", 
        "friends" : [
                {
                        "id" : 0,
                        "name" : "Clarissa Jones"
                },
                {
                        "id" : 1,
                        "name" : "Macias Riley"
                },
                {
                        "id" : 2,
                        "name" : "Dennis Randolph"
                }
            ]
        }

* Запрос через порядковый номер:
	* db.users.aggregate([{$skip: Math.ceil(avg_common["age"]+avg_alaska_age["avg_alaska"])}, {$match: {friends: {$elemMatch: {name: {$regex: /(Dennis )/}}}}}, {$project: {name: 1, friends: 1,address: 1}}, {$limit: 1}]).pretty();
* Ответ:
	* {
        "_id" : ObjectId("5adf3c1544abaca147cdd539"),
        "name" : "Keller Nixon",
        "address" : "591 Jamison Lane, Idamay, Minnesota, 3128",
        "friends" : [
                {
                        "id" : 0,
                        "name" : "Clarissa Jones"
                },
                {
                        "id" : 1,
                        "name" : "Macias Riley"
                },
                {
                        "id" : 2,
                        "name" : "Dennis Randolph"
                }
             ]
	};
	
`Получилось тоже самое, сделал так, потому как до конца не понял что имеется под видом порядковый номер`   
# 4 - Найти активных людей из того же штата, что и предыдущий человек и посмотреть какой фрукт любят больше всего в этом штате (аггрегация) 
`Был получен штат -Minnesota-`

  * Запрос для поиска всех активных людей из штата Minnesota
  	* db.users.aggregate([{$match: { $and: [{address: {$regex: /(Minnesota)/}}, {isActive: true}]}},{$project: {_id: 0, name: 1, address: 1, favoriteFruit: 1}}]).pretty();
  
  * Ответ:
      *  {
            "name" : "Jodie English",
            "address" : "998 Bridgewater Street, Summerfield, Minnesota, 1007",
            "favoriteFruit" : "banana"
          }
          {
            "name" : "Betsy Haynes",
            "address" : "480 Ridgewood Place, Hickory, Minnesota, 4541",
            "favoriteFruit" : "apple"
          }
          {
            "name" : "Riley Salinas",
            "address" : "965 Indiana Place, Wyoming, Minnesota, 4500",
            "favoriteFruit" : "apple"
          }
          {
            "name" : "Belinda Montoya",
            "address" : "174 Louis Place, Glenville, Minnesota, 722",
            "favoriteFruit" : "apple"
          }
          {
            "name" : "Dickerson Lee",
            "address" : "448 Columbia Place, Ticonderoga, Minnesota, 8183",
            "favoriteFruit" : "apple"
          }
          {
            "name" : "Erickson Avery",
            "address" : "969 Fairview Place, Wikieup, Minnesota, 387",
            "favoriteFruit" : "strawberry"
          }
          
  * Запрос какой фрукт любят больше всего в штате Minnesota (аггрегация)
      * db.users.aggregate([{$match: { $and: [{address: {$regex: /(Minnesota)/}}, {isActive: true}]}}, {$group: {_id: "$favoriteFruit", count: {$sum: 1}}}, {$sort: {count: -1}}, {$limit: 1}]);
  
  * Ответ:
      * { "_id" : "apple", "count" : 4 }

# 5 - Найти саммого раннего зарегистрировавшегося пользователя с таким любимым фруктом
  * Запрос:
    * db.users.aggregate([{$match: {favoriteFruit: "apple"}}, {$project: {name: 1, registered: 1,favoriteFruit: 1}}, {$sort: {registered: 1}}, {$limit: 1}]).pretty();
  * Ответ:
    * {
        "_id" : ObjectId("5adf3c1544abaca147cdd568"),
        "name" : "Magdalena Compton",
        "registered" : "2014-01-02T10:16:56 -02:00",
        "favoriteFruit" : "apple"
       }

# 6 - Добавить этому пользовелю свойтво: { features: 'first apple eater' }
  * Запрос:
    * db.users.update({_id: ObjectId("5adf3c1544abaca147cdd568")}, {$set: {features: 'first apple eater'}});
  
  * Ответ:
    * WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
    
  ###### Обновленная сущность:
  * Запрос: 
    * db.users.aggregate([{$match: {favoriteFruit: "apple"}}, {$project: {name: 1, registered: 1, favoriteFruit: 1, features: 1}}, {$sort: {registered: 1}}, {$limit: 1}]).pretty();
  
  * Ответ:
    * {
          "_id" : ObjectId("5adf3c1544abaca147cdd568"),
          "name" : "Magdalena Compton",
          "registered" : "2014-01-02T10:16:56 -02:00",
          "favoriteFruit" : "apple",
          "features" : "first apple eater"
        }
        
# 7 - Удалить всех любителей клубники (написать количество удаленных пользователей)
  * Запрос:
    * db.users.remove({favoriteFruit: {$eq: "strawberry"}});
  * Ответ:
    * WriteResult({ "nRemoved" : 253 }); 
  `Было удалено 253 пользователя, которые любят клубнику` 
