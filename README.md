# Предметная область банкомат для выдачи наличных
# Взаимодействие держателей банковской карточки с банкоматом. Важные сущности - Банковская карточка, пин код, чек, операция смены пин кода, операция перевода средств между своими счетами, операция просмотра остатка на карточке, операция снятия наличных денег, операция оплаты телефона.

class Card:
    def __init__(self, number, pin, balance):
        self.number = number
        self.pin = pin
        self.balance = balance
        self.blocked = False
        self.wrong_pin_count = 0

    def change_pin(self, new_pin):
        self.pin = new_pin

    def transfer(self, amount, card):
        self.balance -= amount
        card.balance += amount

    def check_balance(self):
        return self.balance

    def withdraw(self, amount):
        self.balance -= amount

    def pay_phone(self, amount):
        self.balance -= amount


class ATM:
    def __init__(self, card):
        self.card = card

    def insert_card(self, card):
        self.card = card

    def check_pin(self, pin):
        return self.card.pin == pin

    def change_pin(self, new_pin):
        self.card.change_pin(new_pin)

    def transfer(self, amount, card):
        self.card.transfer(amount, card)

    def check_balance(self):
        return self.card.check_balance()

    def withdraw(self, amount):
        self.card.withdraw(amount)

    def pay_phone(self, amount):
        self.card.pay_phone(amount)


class Receipt:
    def __init__(self, operation, amount, card):
        self.operation = operation
        self.amount = amount
        self.card = card

    def print_receipt(self):
        print(f"Operation: {self.operation}")
        print(f"Amount: {self.amount}")
        print(f"Card: {self.card.number}")
        print(f"Balance: {self.card.balance}")

        return self.amount


class ChangePinOperation:
    def __init__(self, atm, new_pin):
        self.atm = atm
        self.new_pin = new_pin

    def execute(self):
        self.atm.change_pin(self.new_pin)
        return Receipt("Change pin", 0, self.atm.card)


class TransferOperation:
    def __init__(self, atm, amount, card):
        self.atm = atm
        self.amount = amount
        self.card = card

    def execute(self):
        self.atm.transfer(self.amount, self.card)
        return Receipt("Transfer", self.amount, self.atm.card)


class CheckBalanceOperation:
    def __init__(self, atm):
        self.atm = atm

    def execute(self):
        return Receipt("Check balance", self.atm.check_balance(), self.atm.card)


class WithdrawOperation:
    def __init__(self, atm, amount):
        self.atm = atm
        self.amount = amount

    def execute(self):
        self.atm.withdraw(self.amount)
        return Receipt("Withdraw", self.amount, self.atm.card)


class PayPhoneOperation:
    def __init__(self, atm, amount, phone_number):
        self.atm = atm
        self.amount = amount
        self.phone_number = phone_number

    def execute(self):
        if len(self.phone_number) == 12:
            self.atm.pay_phone(self.amount)
            return Receipt("Pay phone", self.amount, self.atm.card)
        else:
            return None


class Operation:
    def __init__(self, atm, operation):
        self.atm = atm
        self.operation = operation

    def execute(self):
        return self.operation.execute()


class Client:
    def __init__(self, card, atm):
        self.card = card
        self.blocked = card.blocked
        self.atm = atm

    def insert_card(self):
        self.atm.insert_card(self.card)

    def check_pin(self, pin):
        return self.atm.check_pin(pin)

    def change_pin(self, new_pin):
        operation = ChangePinOperation(self.atm, new_pin)
        return Operation(self.atm, operation).execute()

    def transfer(self, amount, card):
        operation = TransferOperation(self.atm, amount, card)
        return Operation(self.atm, operation).execute()

    def check_balance(self):
        operation = CheckBalanceOperation(self.atm)
        return Operation(self.atm, operation).execute()

    def withdraw(self, amount):
        operation = WithdrawOperation(self.atm, amount)
        return Operation(self.atm, operation).execute()

    def pay_phone(self, amount, phone_number):
        operation = PayPhoneOperation(self.atm, amount, phone_number)
        return Operation(self.atm, operation).execute()


class Bank:
    def __init__(self):
        self.cards = []

    def add_card(self, card):
        self.cards.append(card)

    def get_card(self, number):
        for card in self.cards:
            if card.number == number:
                return card
        return None

    def get_client(self, number, pin):
        card = self.get_card(number)
        if card is None:
            return None
        if card.pin != pin:
            card.wrong_pin_count += 1
            card.blocked = True if card.wrong_pin_count >= 3 else False

            if card.blocked:
                return card

            return None

        atm = ATM(card)
        return Client(card, atm)


class ATMSimulation:
    def __init__(self, bank):
        self.bank = bank
        self.card_number = None

    def is_enough_money(self, amount):
        return self.card_number.check_balance().amount <= amount

    def get_client(self, number, pin):
        self.card_number = self.bank.get_client(number, pin)

    def run(self):
        while True:
            print("1. вставить карту")
            print("2. Сменить ПИН-КОД")
            print("3. Переводы")
            print("4. Проверить баланс")
            print("5. Снять деньги")
            print("6. Оплатить телефон")
            print("7. Выход")
            choice = input("Введите свой выбор: ")
            if choice == "1":
                number = input("Введите номер карты: ")
                pin = input("Введите пин-код: ")
                self.get_client(number, pin)
                if self.card_number is None:
                    print("Неверный номер карты или пин-код")
                elif self.card_number.blocked:
                    print("Карта заблокирована")
                else:
                    self.card_number.insert_card()
                    print("Карта вставлена")
            elif choice == "2":
                new_pin = input("Введите новый пин-код: ")
                receipt = self.card_number.change_pin(new_pin)
                receipt.print_receipt()
            elif choice == "3":
                number = input("Введите номер карты: ")
                amount = int(input("Введите сумму: "))
                card = self.bank.get_card(number)
                receipt = self.card_number.transfer(amount, card)
                if self.is_enough_money(amount):
                    print("Недостаточно средств")
                else:
                    receipt.print_receipt()
            elif choice == "4":
                receipt = self.card_number.check_balance()
                receipt.print_receipt()
            elif choice == "5":
                print("1. 5 rub")
                print("2. 10 rub")
                print("3. 50 rub")
                print("4. 100 rub")
                print("5. 200 rub")
                print("6. 500 rub")
                select = int(input("Введите вариант: "))
                amount = [5, 10, 50, 100, 200, 500][select - 1]
                if amount is None:
                    print("Неверный вариант")
                elif self.is_enough_money(amount):
                    print("Недостаточно средств")
                else:
                    receipt = self.card_number.withdraw(amount)
                    receipt.print_receipt()
            elif choice == "6":
                phone = input("Введите номер телефона: ")
                amount = int(input("Введите сумму: "))
                receipt = self.card_number.pay_phone(amount, phone)
                if self.is_enough_money(amount):
                    print("Недостаточно средств")
                else:
                    receipt.print_receipt() if receipt is not None else print("Номер телефона не найден")
            elif choice == "7":
                break
            else:
                print("Неверный выбор")


class Main:
    def main(self):
        bank = Bank()
        card1 = Card("1234", "1234", 1000)
        card2 = Card("2345", "2345", 2000)
        bank.add_card(card1)
        bank.add_card(card2)
        simulation = ATMSimulation(bank)
        simulation.run()


if __name__ == "__main__":
    Main().main()

