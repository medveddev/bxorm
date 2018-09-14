# Коллекции

- [Класс](#class)
- [Доступ к элементам](#acc)
  -	[foreach](#acc_foreach)
  -	[getAll / getByPrimary](#acc_get)
  -	[has / hasByPrimary](#acc_has)
  -	[add / []](#acc_add)
  -	[remove, removeByPrimary](#acc_remove)
- [Групповые действия](#group)
  -	[fill](#group_fill)
  -	[get*List](#group_getlist)
- [Восстановление](#wakeup)
  -	[wakeUp](#wakeup)
		

Коллекция — это умный массив, позволяющий оптимизировать групповые операции с объектами одного типа.

Как и в случае с [Объектами](70_objects.md), для начала использования Коллекции объектов достаточно иметь лишь описанную сущность. Используйте `fetchCollection` вместо привычного `fetchAll` или зацикленного `fetch`:

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();

`$books` - коллекция объектов класса `Book`, наделенная методами для групповых манипуляций с объектами. Также коллекции используются в [Отношениях](80_relations_new.md).

<a name='class'></a>
#### Класс коллекции

Действует абсолютно [та же логика](70_objects.md#class) с префиксом `EO_`, что и у Объектов. У каждой сущности свой класс Коллекции, унаследованный от `Bitrix\Main\ORM\Objectify\Collection`. Для сущности `Book` по умолчанию он будет иметь вид `EO_Book_Collection`. Чтобы задать свой класс, нужно создать наследника этого класса и обозначить его в классе Table сущности:

> bitrix/modules/main/lib/test/typography/books.php

	namespace Bitrix\Main\Test\Typography;

    class Books extends EO_Book_Collection
	{
	}

> bitrix/modules/main/lib/test/typography/booktable.php

	namespace Bitrix\Main\Test\Typography;

	class BookTable extends Bitrix\Main\ORM\Data\DataManager
    {
    	public static function getCollectionClass()
		{
			return Books::class;
		}
    	//...
	}

Теперь метод `fetchCollection` будет возвращать коллекцию `Bitrix\Main\Test\Typograph\Books` объектов класса `Bitrix\Main\Test\Typograph\Book`. [Аннотации](90_annotate.md) позволят IDE давать подсказки, облегчая работу разработчика.

<a name='acc'></a>
#### Доступ к элементам коллекции

<a name='acc_foreach'></a>
###### foreach

Базовый класс коллекции реализует интерфейс `\Iterator`, что позволяет перебирать элементы в цикле:

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();

	foreach ($books as $book)
	{
		// ...
	}

<a name='acc_get'></a>
###### getAll, getByPrimary

Объекты коллекции можно получить не только в цикле, но и напрямую. Метод `getAll` вернет все содержащиеся объекты в виде массива:

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();

	$bookObjects = $books->getAll();

	echo $bookObjects[0]->getId();
	// выведет значение ID первого объекта

Для получения конкретных объектов, содержащихся в коллекции, предусмотрен метод `getByPrimary`:

	// 1. пример с простым первичным ключном
	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
			->fetchCollection();
		
	$book = $books->getByPrimary(1);
	// книга с ID=1

	// 2. пример с составным первичным ключом
	$booksToAuthor = \Bitrix\Main\Test\Typography\BookAuthorTable::getList()
		->fetchCollection();

	$bookToAuthor = $booksToAuthor->getByPrimary(
		['BOOK_ID' => 2, 'AUTHOR_ID' => 18]
	);
	// будет присвоен объект отношения книги ID=2 с автором ID=18

<a name='acc_has'></a>
###### has, hasByPrimary

Проверить наличие конкретного объекта в коллекции можно методом `has`:

	$book1 = \Bitrix\Main\Test\Typography\Book::wakeUp(1);
	$book2 = \Bitrix\Main\Test\Typography\Book::wakeUp(2);

	$books = \Bitrix\Main\Test\Typography\BookTable::query()
		->addSelect('*')
		->whereIn('ID', [2, 3, 4])
		->fetchCollection();

	var_dump($books->has($book1));
	// выведет false

	var_dump($books->has($book2));
	// выведет true

Аналогично работает метод `hasByPrimary`, когда удобнее сделать проверку по первичному ключу:

	$books = \Bitrix\Main\Test\Typography\BookTable::query()
		->addSelect('*')
		->whereIn('ID', [2, 3, 4])
		->fetchCollection();

	var_dump($books->hasByPrimary(1));
	// выведет false

	var_dump($books->hasByPrimary(2));
	// выведет true

<a name='acc_add'></a>
###### add, []

Добавление объектов реализовано методом `add` и интерфейсом ArrayAccess, позволяющим использовать конструкцию `[]`:

	$book1 = \Bitrix\Main\Test\Typography\Book::wakeUp(1);

	$books = \Bitrix\Main\Test\Typography\BookTable::query()
		->addSelect('*')
		->whereIn('ID', [2, 3, 4])
		->fetchCollection();

	$books->add($book1);
	// или
	$books[] = $book1;

<a name='acc_remove'></a>
###### remove, removeByPrimary

Удалить объект из коллекции можно, задав его явно или указав первичный ключ:

	$book1 = \Bitrix\Main\Test\Typography\Book::wakeUp(1);

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();

	$books->remove($book1);
	// из коллекции удалится книга с ID=1

	$books->removeByPrimary(2);
	// из коллекции удалится книга с ID=2

<a name='group'></a>
#### Групповые действия

<a name='group_fill'></a>
###### fill

Коллекционная операция `fill` является прекрасной альтернативой аналогичной [операции в Объекте](70_objects.md#fill), выполненной в цикле. В случае с циклом количество запросов к базе данных будет равно количеству объектов:

	/** @var \Bitrix\Main\Test\Typography\Book[] $books */
	$books = [
		\Bitrix\Main\Test\Typography\Book::wakeUp(1),
		\Bitrix\Main\Test\Typography\Book::wakeUp(2)
	];

	foreach ($books as $book)
	{
		$book->fill();
		// SELECT ... WHERE ID = ...
		// так делать не надо!
	}

В случае же с коллекцией запрос будет только один:

	$books = new \Bitrix\Main\Test\Typography\Books;
	// или $books = \Bitrix\Main\Test\Typography\BookTable::getEntity()->createCollection();

	$books[] = \Bitrix\Main\Test\Typography\Book::wakeUp(1);
	$books[] = \Bitrix\Main\Test\Typography\Book::wakeUp(2);
	
	$books->fill();
	// SELECT ... WHERE ID IN(1,2)

Как и в случае с объектами, в качестве параметров в `fill` можно передавать массив имен полей для заполнения или маску типа:

	$books->fill(['TITLE', 'PUBLISHER_ID']);
	$books->fill(\Bitrix\Main\ORM\Fields\FieldTypeMask::FLAT);
 
Более подробно возможные значения параметра описаны в [fill Объектов](70_objects.md#fill).

<a name='group_getlist'></a>
###### get*List

Не самый редкий сценарий - получить из результата запроса список значений отдельного поля. В обычном случае это может выглядеть так:

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();
	
	$titles = [];
	
	foreach ($books as $book)
	{
		$titles[] = $book->getTitle();
	}

Именованный групповой геттер позволяет сократить такой цикл до одной строчки кода:

	$books = \Bitrix\Main\Test\Typography\BookTable::getList()
		->fetchCollection();

	$titles = $books->getTitleList();

Такие геттеры доступны для всех полей сущности и описываются в [аннотациях для IDE](90_annotate.md).


<a name='wakeup'></a>
#### Восстановление коллекции

Восстановление коллекции из готовых данных работает так же, как это [сделано в Объектах](70_objects.md#wakeup):

	// восстановление по первичному ключу
	$books = \Bitrix\Main\Test\Typography\Books::wakeUp([1, 2]);

	// восстановление по набору полей
	$books = \Bitrix\Main\Test\Typography\Books::wakeUp([
		['ID' => 1, 'TITLE' => 'Title 1'],
		['ID' => 2, 'TITLE' => 'Title 2']
	]);

С той лишь разницей, что в коллекцию передается массив с данными объектов, которые нужно разместить в коллекции.