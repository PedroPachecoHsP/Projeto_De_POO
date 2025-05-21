# Projeto de POO da matéria da faculdade: um sistema bancário em Python com banco de dados.


# Explicação do código















# CÓDIGO COMEÇA A PARTIR DESTE PONTO!!!!

import tkinter as tk
from tkinter import messagebox, simpledialog
import sqlite3

# - Banco de Dados -
class BancoDeDados:
    def __init__(self, nome_db="banco.db"):
        self.conn = sqlite3.connect(nome_db)
        self.cursor = self.conn.cursor()
        self.criar_tabelas()

    def criar_tabelas(self):
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS clientes (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                nome TEXT NOT NULL,
                cpf TEXT NOT NULL UNIQUE
            )
        """)
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS contas (
                numero INTEGER PRIMARY KEY,
                cliente_id INTEGER,
                saldo REAL DEFAULT 0,
                FOREIGN KEY (cliente_id) REFERENCES clientes(id)
            )
        """)
        self.cursor.execute("""
            CREATE TABLE IF NOT EXISTS extratos (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                numero_conta INTEGER,
                operacao TEXT,
                valor REAL,
                data TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (numero_conta) REFERENCES contas(numero)
            )
        """)
        self.conn.commit()

    def cpf_existe(self, cpf):
        self.cursor.execute("SELECT id FROM clientes WHERE cpf = ?", (cpf,))
        return self.cursor.fetchone() is not None

    def inserir_cliente(self, nome, cpf):
        if self.cpf_existe(cpf):
            return None
        self.cursor.execute("INSERT INTO clientes (nome, cpf) VALUES (?, ?)", (nome, cpf))
        self.conn.commit()
        return self.cursor.lastrowid

    def inserir_conta(self, numero, cliente_id):
        self.cursor.execute("INSERT INTO contas (numero, cliente_id) VALUES (?, ?)", (numero, cliente_id))
        self.conn.commit()

    def atualizar_saldo(self, numero, novo_saldo):
        self.cursor.execute("UPDATE contas SET saldo = ? WHERE numero = ?", (novo_saldo, numero))
        self.conn.commit()

    def registrar_extrato(self, numero_conta, operacao, valor):
        self.cursor.execute(
            "INSERT INTO extratos (numero_conta, operacao, valor) VALUES (?, ?, ?)",
            (numero_conta, operacao, valor)
        )
        self.conn.commit()

    def buscar_conta_com_cliente(self, numero):
        self.cursor.execute("""
            SELECT contas.numero, contas.saldo, clientes.nome, clientes.cpf
            FROM contas
            JOIN clientes ON contas.cliente_id = clientes.id
            WHERE contas.numero = ?
        """, (numero,))
        return self.cursor.fetchone()

    def obter_extrato(self, numero_conta):
        self.cursor.execute("""
            SELECT operacao, valor, data
            FROM extratos
            WHERE numero_conta = ?
            ORDER BY data ASC
        """, (numero_conta,))
        return self.cursor.fetchall()

    def buscar_todas_contas(self):
        self.cursor.execute("""
            SELECT contas.numero, clientes.nome, clientes.cpf, contas.saldo
            FROM contas
            JOIN clientes ON contas.cliente_id = clientes.id
            ORDER BY contas.numero
        """)
        return self.cursor.fetchall()

    def excluir_cliente_por_cpf(self, cpf):
        # Primeiro pegar o cliente
        self.cursor.execute("SELECT id FROM clientes WHERE cpf = ?", (cpf,))
        cliente = self.cursor.fetchone()
        if not cliente:
            return False
        cliente_id = cliente[0]
        # Apagar extratos das contas do cliente
        self.cursor.execute("SELECT numero FROM contas WHERE cliente_id = ?", (cliente_id,))
        contas = self.cursor.fetchall()
        for (numero_conta,) in contas:
            self.cursor.execute("DELETE FROM extratos WHERE numero_conta = ?", (numero_conta,))
        # Apagar contas
        self.cursor.execute("DELETE FROM contas WHERE cliente_id = ?", (cliente_id,))
        # Apagar cliente
        self.cursor.execute("DELETE FROM clientes WHERE id = ?", (cliente_id,))
        self.conn.commit()
        return True

# - Classes de dados com encapsulamento -
class Cliente:
    def __init__(self, nome, cpf):
        self.__nome = nome
        self.__cpf = cpf

    @property
    def nome(self):
        return self.__nome

    @property
    def cpf(self):
        return self.__cpf

class Conta:
    def __init__(self, numero, cliente, saldo=0.0, extrato=None):
        self.__numero = numero
        self.__cliente = cliente
        self.__saldo = saldo
        self.__extrato = extrato if extrato else []

    @property
    def numero(self):
        return self.__numero

    @property
    def cliente(self):
        return self.__cliente

    @property
    def saldo(self):
        return self.__saldo

    @saldo.setter
    def saldo(self, valor):
        self.__saldo = valor

    @property
    def extrato(self):
        return self.__extrato

    @extrato.setter
    def extrato(self, valor):
        self.__extrato = valor

    def formatar_extrato(self):
        if not self.__extrato:
            return "Nenhuma movimentação encontrada."
        linhas = []
        for op, val, data in self.__extrato:
            linhas.append(f"{data[:16]} - {op}: R$ {val:.2f}")
        return "\n".join(linhas)

# - Interface Gráfica -
class BancoGUI:
    def __init__(self, root):
        self.root = root
        self.root.title("🏦 Banco Seguro")
        self.root.geometry("500x500")
        self.root.configure(bg="#f0f0f0")
        self.db = BancoDeDados()

        tk.Label(root, text="🏦 Banco Seguro", font=("Arial", 20, "bold"), bg="#f0f0f0").pack(pady=20)

        botoes = [
            ("➕ Criar Cliente e Conta", self.criar_cliente),
            ("💰 Depositar", self.depositar),
            ("💸 Sacar", self.sacar),
            ("📋 Ver Extrato", self.ver_extrato),
            ("🗑️ Excluir Cliente", self.excluir_cliente),
            ("📊 Mostrar Todas as Contas", self.mostrar_todas_contas),
        ]

        for texto, comando in botoes:
            tk.Button(root, text=texto, command=comando, font=("Arial", 12), width=35,
                      bg="#4CAF50", fg="white", bd=0, pady=8).pack(pady=5)

    def pedir_numero_conta(self):
        entrada = simpledialog.askstring("Conta", "Número da conta:")
        if entrada and entrada.isdigit():
            return int(entrada)
        messagebox.showerror("Erro", "Número inválido.")
        return None

    def pedir_cpf(self):
        cpf = simpledialog.askstring("CPF", "Digite o CPF do cliente:")
        if cpf:
            return cpf
        messagebox.showerror("Erro", "CPF inválido.")
        return None

    def buscar_conta(self, numero):
        dados = self.db.buscar_conta_com_cliente(numero)
        if dados:
            numero, saldo, nome, cpf = dados
            cliente = Cliente(nome, cpf)
            extrato = self.db.obter_extrato(numero)
            return Conta(numero, cliente, saldo, extrato)
        return None

    def criar_cliente(self):
        nome = simpledialog.askstring("Nome", "Nome do cliente:")
        cpf = simpledialog.askstring("CPF", "CPF do cliente:")
        numero_str = simpledialog.askstring("Conta", "Número da nova conta:")

        if not (nome and cpf and numero_str and numero_str.isdigit()):
            messagebox.showerror("Erro", "Dados inválidos.")
            return

        numero = int(numero_str)

        if self.db.cpf_existe(cpf):
            messagebox.showerror("Erro", "CPF já cadastrado.")
            return

        cliente_id = self.db.inserir_cliente(nome, cpf)
        if cliente_id is None:
            messagebox.showerror("Erro", "Erro ao criar cliente.")
            return

        try:
            self.db.inserir_conta(numero, cliente_id)
        except sqlite3.IntegrityError:
            messagebox.showerror("Erro", "Número da conta já existe.")
            return

        messagebox.showinfo("Sucesso", f"Cliente {nome} cadastrado com a conta {numero}.")

    def depositar(self):
        numero = self.pedir_numero_conta()
        if numero is None: return

        conta = self.buscar_conta(numero)
        if conta:
            valor_str = simpledialog.askstring("Depósito", "Valor a depositar:")
            try:
                valor = float(valor_str)
                if valor <= 0:
                    raise ValueError
            except:
                messagebox.showerror("Erro", "Valor inválido!")
                return

            conta.saldo += valor
            self.db.atualizar_saldo(numero, conta.saldo)
            self.db.registrar_extrato(numero, "Depósito", valor)
            messagebox.showinfo("Sucesso", f"Depósito de R$ {valor:.2f} realizado.")
        else:
            messagebox.showerror("Erro", "Conta não encontrada.")

    def sacar(self):
        numero = self.pedir_numero_conta()
        if numero is None: return

        conta = self.buscar_conta(numero)
        if conta:
            valor_str = simpledialog.askstring("Saque", "Valor a sacar:")
            try:
                valor = float(valor_str)
                if valor <= 0 or valor > conta.saldo:
                    raise ValueError
            except:
                messagebox.showerror("Erro", "Saldo insuficiente ou valor inválido!")
                return

            conta.saldo -= valor
            self.db.atualizar_saldo(numero, conta.saldo)
            self.db.registrar_extrato(numero, "Saque", valor)
            messagebox.showinfo("Sucesso", f"Saque de R$ {valor:.2f} realizado.")
        else:
            messagebox.showerror("Erro", "Conta não encontrada.")

    def ver_extrato(self):
        numero = self.pedir_numero_conta()
        if numero is None: return

        conta = self.buscar_conta(numero)
        if conta:
            extrato_formatado = conta.formatar_extrato()
            mensagem = f"{extrato_formatado}\n\nSaldo atual: R$ {conta.saldo:.2f}"
            messagebox.showinfo("Extrato", mensagem)
        else:
            messagebox.showerror("Erro", "Conta não encontrada.")

    def excluir_cliente(self):
        cpf = self.pedir_cpf()
        if not cpf:
            return

        if not self.db.cpf_existe(cpf):
            messagebox.showerror("Erro", "CPF não cadastrado.")
            return

        confirmar = messagebox.askyesno("Confirmação", f"Tem certeza que deseja excluir o cliente com CPF {cpf} e todas as suas contas?")
        if confirmar:
            sucesso = self.db.excluir_cliente_por_cpf(cpf)
            if sucesso:
                messagebox.showinfo("Sucesso", "Cliente e contas excluídos.")
            else:
                messagebox.showerror("Erro", "Erro ao excluir cliente.")

    def mostrar_todas_contas(self):
        contas = self.db.buscar_todas_contas()
        if not contas:
            messagebox.showinfo("Contas Cadastradas", "Nenhuma conta cadastrada.")
            return

        janela = tk.Toplevel(self.root)
        janela.title("Todas as Contas Cadastradas")
        janela.geometry("500x300")
        janela.configure(bg="#f0f0f0")

        tk.Label(janela, text="Contas Cadastradas", font=("Arial", 16, "bold"), bg="#f0f0f0").pack(pady=10)

        frame_texto = tk.Frame(janela)
        frame_texto.pack(fill="both", expand=True, padx=10, pady=10)

        scrollbar = tk.Scrollbar(frame_texto)
        scrollbar.pack(side="right", fill="y")

        texto = tk.Text(frame_texto, yscrollcommand=scrollbar.set, font=("Courier New", 11))
        texto.pack(fill="both", expand=True)
        scrollbar.config(command=texto.yview)

        texto.insert("end", f"{'Conta':<10} {'Cliente':<20} {'CPF':<15} {'Saldo':>10}\n")
        texto.insert("end", "-"*60 + "\n")
        for numero, nome, cpf, saldo in contas:
            texto.insert("end", f"{numero:<10} {nome:<20} {cpf:<15} R$ {saldo:>9.2f}\n")

        texto.config(state="disabled")

# - Rodar App -
if __name__ == "__main__":
    root = tk.Tk()
    app = BancoGUI(root)
    root.mainloop()
