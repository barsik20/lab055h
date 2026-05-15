# Homework
[![Coverage Status](https://coveralls.io/repos/github/barsik20/lab055h/badge.svg)](https://coveralls.io/github/barsik20/lab055h)

Сначала написал CMakeLists в папке с исходниками. Там я собираю две статические библиотеки — account и transaction — отдельно, чтобы тесты подтягивали только нужное.
```cmake
cmake_minimum_required(VERSION 3.10)
project(banking)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if (NOT TARGET account)
	add_library(account STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Account.cpp)
endif()

if (NOT TARGET transaction)
	add_library(transaction STATIC ${CMAKE_CURRENT_SOURCE_DIR}/Transaction.cpp)
endif()

target_include_directories(account PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_include_directories(transaction PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_directories(transaction PUBLIC account)

if (COLLECT_COVERAGE)
	target_compile_options(account PRIVATE -O0 -g --coverage)
	target_link_options(account PRIVATE --coverage)
	target_compile_options(transaction PRIVATE -O0 -g --coverage)
	target_link_options(transaction PRIVATE --coverage)
endif()
```

Главный CMakeLists.txt
В корневом файле я подключаю поддиректорию banking и, если включена опция BUILD_TESTS, собираю тесты с gtest и gmock.
```cmake
cmake_minimum_required(VERSION 3.10)
project(banking_tests)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

option(BUILD_TESTS "Build tests" OFF)
option(COLLECT_COVERAGE "Collect coverage" OFF)

add_subdirectory(banking)

if (BUILD_TESTS)
	enable_testing()
	add_subdirectory(third-party/gtest)
	
	file(GLOB TEST_SOURCES "${CMAKE_CURRENT_SOURCE_DIR}/tests/*.cpp")
	add_executable(check ${TEST_SOURCES})
	target_link_libraries(check account transaction gtest_main gmock_main)

	if (COLLECT_COVERAGE)
		target_link_libraries(check gcov)
		target_compile_options(check PRIVATE -O0 -g --coverage)
		target_link_options(check PRIVATE --coverage)
	endif()

	add_test(NAME check COMMAND check)
endif()
```
##Тесты для класса Account
Для Account я использовал фикстуру (AccountFixture), чтобы не создавать объект в каждом тесте заново. Это удобно: SetUp создаёт аккаунт, TearDown чистит память (кстати, деструктор тоже неявно проверяется).
```cpp
#include <Account.h>
#include <gtest/gtest.h>

class AccountFixture : public testing::Test {
public:
	Account* acc;
	void SetUp() { acc = new Account(123, 1000); }
	void TearDown() { delete acc; }
};

TEST_F(AccountFixture, GetBalance) {
	EXPECT_EQ(acc->GetBalance(), 1000);
}

TEST_F(AccountFixture, ChangeBalanceGood) {
	acc->Lock();
	acc->ChangeBalance(200);
	EXPECT_EQ(acc->GetBalance(), 1200);
}

TEST_F(AccountFixture, ChangeBalanceBad) {
	EXPECT_THROW(acc->ChangeBalance(100), std::runtime_error);
}

TEST_F(AccountFixture, GetID) {
	EXPECT_EQ(acc->id(), 123);
}

TEST_F(AccountFixture, LockTwice) {
	acc->Lock();
	EXPECT_THROW(acc->Lock(), std::runtime_error);
}

TEST_F(AccountFixture, LockAndUnlock) {
	acc->Lock();
	acc->Unlock();
	EXPECT_NO_THROW(acc->Lock());
}
```
Это мы создали Account
Далее создаём Transaction
```cpp
#include <Account.h>
#include <Transaction.h>
#include <gtest/gtest.h>
#include <gmock/gmock.h>

class AccountMock : public Account {
public:
	AccountMock(int id, int balance) : Account(id, balance) {}
	
	MOCK_METHOD(int, GetBalance, (), (const, override));
  	MOCK_METHOD(void, ChangeBalance, (int diff), (override));
  	MOCK_METHOD(void, Lock, (), (override));
  	MOCK_METHOD(void, Unlock, (), (override));
};

class TransactionFixture : public testing::Test {
public:
	Transaction* tr;
	AccountMock* from;
	AccountMock* to;
	void SetUp () override { 
		tr = new Transaction;
		from = new testing::NiceMock<AccountMock>(1, 1000);
		to = new testing::NiceMock<AccountMock>(2, 1000);
	}
	void TearDown () override { 
		delete tr;
		delete from;
		delete to;
	}
};

TEST_F(TransactionFixture, Fee) {
	EXPECT_EQ(tr->fee(), 1);
}

TEST_F(TransactionFixture, SetFee) {
	tr->set_fee(10);
	EXPECT_EQ(tr->fee(), 10);
}

TEST_F(TransactionFixture, SuccessfulTransfer) {
	EXPECT_CALL(*to, ChangeBalance(200)).Times(1);
	EXPECT_CALL(*from, GetBalance()).WillOnce(testing::Return(1000)).WillRepeatedly(testing::Return(799));
	EXPECT_CALL(*to, GetBalance()).WillRepeatedly(testing::Return(1200));

	EXPECT_TRUE(tr->Make(*from, *to, 200));
}

TEST_F(TransactionFixture, TransferToYourself) {
	EXPECT_THROW(tr->Make(*from, *from, 200), std::logic_error);
}

TEST_F(TransactionFixture, NegativeSumTransfer) {
	EXPECT_THROW(tr->Make(*from, *to, -100), std::invalid_argument);
}

TEST_F(TransactionFixture, TooSmallSumTransfer) {
	EXPECT_THROW(tr->Make(*from, *to, 70), std::logic_error);
}

TEST_F(TransactionFixture, TooBigFee) {
	tr->set_fee(60);
	EXPECT_FALSE(tr->Make(*from, *to, 100));
}

TEST_F(TransactionFixture, TooBigSumTransfer) {
	EXPECT_CALL(*from, GetBalance()).WillRepeatedly(testing::Return(1000));
	EXPECT_CALL(*to, ChangeBalance(1200)).Times(1);
	EXPECT_CALL(*to, ChangeBalance(-1200)).Times(1);
	EXPECT_CALL(*to, GetBalance()).WillRepeatedly(testing::Return(1000));
	EXPECT_FALSE(tr->Make(*from, *to, 1200));
}
```
Самый интересный момент возник при тестировании Transaction. Поскольку этот класс напрямую работает с Account, между ними образовалась тесная связь. Если бы я тестировал их вместе, получилась бы "матрешка" — чтобы проверить Transaction, мне пришлось бы одновременно отлаживать и Account. Это неудобно и нарушает принципы модульного тестирования. Поэтому я применил mock-объекты. Благодаря им можно не вникать во внутренности Account, а просто имитировать его поведение и сосредоточиться исключительно на логике Transaction. Сначала я создал класс-заглушку AccountMock, унаследовав его от Account. Все виртуальные методы я объявил через макросы Google Mock (кроме метода id — он не виртуальный, поэтому подменить его не выйдет). Затем в фикстуре для Transaction я использовал два таких мока вместо реальных аккаунтов. Так как заглушки сами по себе ничего не делают, мне пришлось явно описать их поведение с помощью макроса EXPECT_CALL. По сути, я искусственно создал "идеальный" аккаунт, сам задавая его реакции, и на этом фоне проверил, как Transaction обрабатывает переводы, комиссии и ошибки.

Вот вывод build
```
root@DESKTOP-BAMMTI8:~/barsik20/workspace/projects/lab055h# cmake --build _build
[  6%] Building CXX object third-party/gtest/googletest/CMakeFiles/gtest.dir/src/gtest-all.cc.o
[ 13%] Linking CXX static library ../../../lib/libgtest.a
[ 13%] Built target gtest
[ 20%] Building CXX object third-party/gtest/googletest/CMakeFiles/gtest_main.dir/src/gtest_main.cc.o
[ 26%] Linking CXX static library ../../../lib/libgtest_main.a
[ 26%] Built target gtest_main
[ 33%] Building CXX object banking/CMakeFiles/account.dir/Account.cpp.o
[ 40%] Linking CXX static library libaccount.a
[ 40%] Built target account
[ 46%] Building CXX object banking/CMakeFiles/transaction.dir/Transaction.cpp.o
[ 53%] Linking CXX static library libtransaction.a
[ 53%] Built target transaction
[ 60%] Building CXX object third-party/gtest/googlemock/CMakeFiles/gmock.dir/src/gmock-all.cc.o
[ 66%] Linking CXX static library ../../../lib/libgmock.a
[ 66%] Built target gmock
[ 73%] Building CXX object third-party/gtest/googlemock/CMakeFiles/gmock_main.dir/src/gmock_main.cc.o
[ 80%] Linking CXX static library ../../../lib/libgmock_main.a
[ 80%] Built target gmock_main
[ 86%] Building CXX object CMakeFiles/check.dir/tests/AccountTest.cpp.o
[ 93%] Building CXX object CMakeFiles/check.dir/tests/TransactionTest.cpp.o
[100%] Linking CXX executable check
[100%] Built target check
```
Вывод _build/check
```
root@DESKTOP-BAMMTI8:~/barsik20/workspace/projects/lab055h# _build/check
Running main() from /root/barsik20/workspace/projects/lab055h/third-party/gtest/googletest/src/gtest_main.cc
[==========] Running 14 tests from 2 test suites.
[----------] Global test environment set-up.
[----------] 6 tests from AccountFixture
[ RUN      ] AccountFixture.GetBalance
[       OK ] AccountFixture.GetBalance (0 ms)
[ RUN      ] AccountFixture.ChangeBalanceGood
[       OK ] AccountFixture.ChangeBalanceGood (0 ms)
[ RUN      ] AccountFixture.ChangeBalanceBad
[       OK ] AccountFixture.ChangeBalanceBad (0 ms)
[ RUN      ] AccountFixture.GetID
[       OK ] AccountFixture.GetID (0 ms)
[ RUN      ] AccountFixture.LockTwice
[       OK ] AccountFixture.LockTwice (0 ms)
[ RUN      ] AccountFixture.LockAndUnlock
[       OK ] AccountFixture.LockAndUnlock (0 ms)
[----------] 6 tests from AccountFixture (1 ms total)

[----------] 8 tests from TransactionFixture
[ RUN      ] TransactionFixture.Fee
[       OK ] TransactionFixture.Fee (0 ms)
[ RUN      ] TransactionFixture.SetFee
[       OK ] TransactionFixture.SetFee (0 ms)
[ RUN      ] TransactionFixture.SuccessfulTransfer
1 send to 2 $200
Balance 1 is 799
Balance 2 is 1200
[       OK ] TransactionFixture.SuccessfulTransfer (0 ms)
[ RUN      ] TransactionFixture.TransferToYourself
[       OK ] TransactionFixture.TransferToYourself (0 ms)
[ RUN      ] TransactionFixture.NegativeSumTransfer
[       OK ] TransactionFixture.NegativeSumTransfer (0 ms)
[ RUN      ] TransactionFixture.TooSmallSumTransfer
[       OK ] TransactionFixture.TooSmallSumTransfer (0 ms)
[ RUN      ] TransactionFixture.TooBigFee
[       OK ] TransactionFixture.TooBigFee (0 ms)
[ RUN      ] TransactionFixture.TooBigSumTransfer
1 send to 2 $1200
Balance 1 is 1000
Balance 2 is 1000
[       OK ] TransactionFixture.TooBigSumTransfer (0 ms)
[----------] 8 tests from TransactionFixture (0 ms total)

[----------] Global test environment tear-down
[==========] 14 tests from 2 test suites ran. (2 ms total)
[  PASSED  ] 14 tests.
```
Все тесты и сборка сделаны.
Просмотр покрытия кода:
```
root@DESKTOP-BAMMTI8:~/barsik20/workspace/projects/lab055h# genhtml cov.info --output-directory coverage_report
Found 2 entries.
Found common filename prefix "/root/barsik20/workspace/projects/lab055h"
Generating output.
Processing file banking/Transaction.cpp
  lines=33 hit=33 functions=9 hit=9
Processing file banking/Account.cpp
  lines=16 hit=16 functions=8 hit=8
Overall coverage rate:
  lines......: 100.0% (49 of 49 lines)
  functions......: 100.0% (17 of 17 functions)
```


```
root@DESKTOP-BAMMTI8:~/barsik20/workspace/projects/lab055h# lcov --capture --directory _build/banking --output-file=cov.info
Capturing coverage data from _build/banking
geninfo cmd: '/usr/bin/geninfo _build/banking --output-filename cov.info --memory 0'
Found gcov version: 13.3.0
Using intermediate gcov format
Writing temporary data to /tmp/geninfo_datEcA6
Scanning _build/banking for .gcda files ...
Found 2 data files in _build/banking
Processing _build/banking/CMakeFiles/transaction.dir/Transaction.cpp.gcda
Processing _build/banking/CMakeFiles/account.dir/Account.cpp.gcda
Finished .info-file creation
```

```
root@DESKTOP-BAMMTI8:~/barsik20/workspace/projects/lab055h# lcov --list cov.info
                      |Lines       |Functions  |Branches
Filename              |Rate     Num|Rate    Num|Rate     Num
============================================================
[/root/barsik20/workspace/projects/lab055h/banking/]
Account.cpp           |100.0%     16| 100.0%     8|    -      0
Transaction.cpp       |100.0%     33| 100.0%     9|    -      0
============================================================
                Total:|34.7%     49| 0.0%    17|    -      0
```


Создадим скрипт для Github Actions:
```
name: coverage_in_linux
on: push
jobs:
  build_and_test:
    runs-on: ubuntu-latest
    steps:
      - name: Get repo files
        uses: actions/checkout@v6
        with:
          submodules: true
      - name: Configure build directory and options
        run: cmake -H. -B _build -D BUILD_TESTS=ON -D COLLECT_COVERAGE=ON
      - name: Build project
        run: cmake --build _build
      - name: Run tests
        run: ./_build/check
      - name: Install lcov
        run: sudo apt install lcov
      - name: Make coverage configuration file
        run: lcov --capture --directory _build/banking --gcov-tool /usr/bin/gcov-13 --output-file=cov.info
      - name: Get coverage percentage
        run: lcov --list cov.info
      - name: Coveralls Upload
        uses: coverallsapp/github-action@v2
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          file: cov.info
```


Всё собралось и github actions работает
