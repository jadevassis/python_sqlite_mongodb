# python_sqlite_mongodb
Projeto Integrando Python com SQLite e MongoDB

from sqlalchemy import create_engine
from sqlalchemy import func
from sqlalchemy import select
from sqlalchemy import Column, Float, Integer, String, ForeignKey
from sqlalchemy.orm import declarative_base, relationship, Session
import random

Base = declarative_base()


class Client(Base):
    __tablename__ = "clients"
    id = Column(Integer, primary_key=True, autoincrement=True)
    name = Column(String(50), nullable=False)
    ssn = Column(String(11), nullable=False, unique=True)
    address = Column(String(50), nullable=False)

    accounts = relationship(
        "Account", back_populates="client"
    )

    def __repr__(self):
        return f"Client(id={self.id}, name={self.name}, ssn={self.ssn}, address={self.address})"


class Account(Base):
    __tablename__ = "accounts"
    id = Column(Integer, primary_key=True, autoincrement=True)
    type = Column(String, default="Checking Account")
    agency = Column(String, nullable=False)
    number = Column(Integer, nullable=False)
    balance = Column(Float(asdecimal=True), nullable=False, default=0)
    client_id = Column(Integer, ForeignKey("clients.id"), nullable=False)

    client = relationship(
        "Client", back_populates="accounts"
    )

    def __repr__(self):
        return (f"Account(id={self.id}, type={self.type}, agency={self.agency}, number={self.number}, "
                f"balance={self.balance})")


def generate_random_ssn():
    ssn = [str(random.randint(0, 9)) for _ in range(9)]
    ssn.insert(3, "-")
    ssn.insert(6, "-")
    return "".join(ssn)


MOCK_DATA = [
    Client(
        name='Jader Rodrigues',
        ssn='000-00-0000',
        address='Ary Zanella, Jardim Sao Carlos',
        accounts=[
            Account(
                agency='0001',
                number=1342,
                balance=12222,
            )
        ]
    ),
    Client(
        name='Thaina Lopes',
        ssn='000-00-0001',
        address='José Lemos, Celeste',
        accounts=[
            Account(
                agency='0001',
                number=9999,
            ),
            Account(
                type='Savings',
                agency='0001',
                number=51009999,
                balance=300.00,
            )
        ]
    ),
    Client(
        name='Gwen Stacy',
        ssn='000-00-0002',
        address='999 Oak St, Inverted World',
        accounts=[
            Account(
                agency='0001',
                number=4321,
                balance=30,
            )
        ]
    ),
]

engine = create_engine("sqlite+pysqlite:///:memory:")
Base.metadata.create_all(engine)

with Session(engine) as session:
    session.add_all(MOCK_DATA)
    session.commit()

stmts = [
    {
        "msg": "Retrieving clients based on filtering condition",
        "stmt": select(Client).where(Client.name.in_(['John Doe', 'Bob Johnson'])),
    },
    {
        "msg": "Retrieving accounts of Jane Smith",
        "stmt": select(Account).where(Account.client_id.in_([2])),
    },
    {
        "msg": "Retrieving information in an ordered manner",
        "stmt": select(Account).order_by(Account.number.asc()),
    },
    {
        "msg": "Retrieving cross-referenced information",
        "stmt": select(Client.ssn, Account.balance).join_from(Client, Account),
    },
    {
        "msg": "Total instances in Account with positive balance",
        "stmt": select(func.count('*')).select_from(Account).filter(Account.balance > 0),
    },
]

with Session(engine) as session:
    for item in stmts:
        print(f"\n{item['msg']}")
        print(f"\n{item['stmt']}")
        for result in session.scalars(item['stmt']):
            print(result)

    connection = engine.connect()
    results = connection.execute(stmts[3]['stmt']).fetchall()
    print("\nExecuting a statement using a database connection")
    for result in results:
        print(result)
