# Аудит смартконтрактов проекта BitImage

## Уровни выявленных проблем

##### Низкий — не оказывает существенного влияния на возможные сценарии использования контракта, часто бывает субъективной.
##### Средний — уязвимость, которая может повлиять на желаемый результат исполнения контракта в конкретном сценарии.
##### Высокий — уязвимость, которая влияет на желаемый результат при использовании контракта или предоставляет возможность использовать контракт не регламентированным образом.
##### Критический — уязвимость, которая может вызвать нарушение работы контракта по ряду сценариев или предоставляет возможность нарушить работу контракта.


## Анализ файла contracts/BitImageToken.sol

### Низкий уровень

###### 1.

Используется не самая свежая версия библиотеки Zeppelin solidity. В новой добавлено событие `Transfer(burner, address(0), _value);` при сжигании токена по аналогии с https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#transfer-1

###### 2. setSaleAgent

Нет необходимости делать `super.approve(saleAgent, 0);` перед `super.approve(saleAgent, totalSupply);`, если это происходит в одной транзакции

###### 3. burnFrom

По аналогии с https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md#transfer-1 можно выбрасывать событие `Transfer(burner, address(0), _value);`


### Средний уровень

###### 1. setSaleAgent

При переназначении SaleAgent у старого не отнимается возможность распоряжаться токенами owner'а




## Анализ файла contracts/BitImageCrowdsale.sol

### Низкий уровень

###### 1.

Неиспользуемая переменная `investorsIndex`, только записывается значение, видимость private

###### 2. refund

Инвесторы не защищены от того, что владелец может поставить контракт на паузу, и тем самым остановить возврат средств через метод refund.

### Средний уровень

###### 1.

При следующем сценарии незапланированно увеличивается бонус:
- presale завершен, начат crowdsale (бонус 25%)
- в первой неделе достигнут goal (бонус уменьшается до 20%, устанавливается новый goal)
- в этой же неделе достигнут новый goal (бонус уменьшается до 15%, устанавливается новый goal)
- наступает новая неделя, бонус устанавливается в 20% (из-за `bonus = bonusAfterPresale.sub(bonusDicrement.mul(currentWeek));`)

### Высокий уровень

###### 1.

Обычно награда за bounty и для эдвайзеров рассылаются после окончания распродажи. И после этого делается так, чтобы пользователи не могли тратить эти токены, находящиеся уже на их кошельке.

Данный контракт замораживает переводы токенов с кошельков, предназначенного для bounty и для эдвайзеров на 180 дней. Т.е. перевести награду пользователям можно будет только через полгода.

### Критический уровень

###### 1.

При следующей последовательности действий деньги, полученные от пользователей, зависнут на контракте без возможности вывести:

- вызов метода `start` - начало presale
- сбор средств (часть денег собрана, но softcap не набран)
- вызов метода `pause` - установка паузы
- вызов метода `start` - начало crowdsale
После этого неважно, будет ли набран softcap или нет, деньги, собранные во время presale, нельзя будет вывести:
- пользователям через метод `refund`, т.к. он работает только при presale
- владельцу, т.к. метод finalize переводит весь эфир из контракта только при presale

Да, это возможно только при определенных действиях владельца контракта, но контракт физически не должен позволять такое сделать.