<h1>Projeto de POO da matéria da faculdade: um sistema bancário em Python com banco de dados.</h1>

<h1>Explicação do código</h1>

<h4>📂1. Banco de Dados – BancoDeDados</h4>
<p>Classe responsável por interagir com o SQLite.</p>

<p><strong> __init__</strong>: conecta ao banco e cria tabelas se não existirem.</p>
<p><strong>Criar_tabelas:</strong><br> Define 3 tabelas:</p>
    <ul>-<strong>clientes:</strong> armazena nome e CPF.</ul>
    <ul>-<strong>contas:</strong> liga um número de conta a um cliente.</ul>  
    <ul>-<strong>extratos:</strong> registra depósitos, saques e suas datas.</ul>
    
<p>Métodos como <strong>(inserir_cliente)</strong>, <strong>(cpf_existe)</strong>, <strong>(atualizar_saldo)</strong>, <strong>(registrar_extrato)</strong> cuidam da lógica de dados.</p>

<p><strong>(Excluir_cliente_por_cpf:)</strong> deleta o cliente, suas contas e extratos (com segurança relacional). </p><br>

<h4>👤2. Classes de Dados – Cliente e Conta</h4>
<p>Usando encapsulamento com <strong>@property.</strong></p>

<p><strong>Cliente:</strong> Guarda <strong>(nome)</strong> e <strong>(cpf)</strong>, com atributos privados <strong>(__nome)</strong> e <strong>(__cpf)</strong>.</p>
<p>Conta: Guarda</p>
    <ul>-<strong>Numero</strong>: da conta.</ul>
    <ul>-<strong>Cliente</strong>: objeto Cliente.</ul>
    <ul>-<strong>Saldo</strong>: valor atual.</ul>
    <ul>-<strong>Extrato</strong>: lista de operações.</ul>
<p><strong>Formatar_extrato</strong>: gera texto com o histórico de transações.</p>

<h4>🏜️ 3. Interface Gráfica – BancoGUI </h4>
<p>Classe principal da <strong>GUI com Tkinter</strong>.</p>

<p>Cria botões para:</p>

<ul>-Criar cliente/conta.</ul>
<ul>-Depositar.</ul>
<ul>-Sacar.</ul>
<ul>-Ver extrato.</ul>
<ul>-Excluir cliente.</ul>

<p>Métodos importantes:</p>

<ul>-<strong>criar_cliente</strong>: cadastra cliente e cria conta.</ul>
<ul>-<strong>depositar</strong>: atualiza saldo e grava no extrato.</ul>
<ul>-<strong>sacar</strong>: verifica saldo e registra saque.</ul>
<ul>-<strong>ver_extrato</strong>: mostra histórico com saldo atual.</ul>
<ul>-<strong>excluir_cliente</strong>: remove tudo com confirmação.</ul>


# O Código Tem Inicio Aqui

import tkinter as tk
from tkinter import messagebox, simpledialog
import sqlite3


# Banco de Dados
class BancoDeDados:
    def __init__(self, db="banco.db"):
        self.conn = sqlite3.connect(db)
        self.conn.execute("PRAGMA foreign_keys = ON")
        self.cursor = self.conn.cursor()
        self.cursor.executescript("""
            CREATE TABLE IF NOT EXISTS clientes (
                id INTEGER PRIMARY KEY,
                nome TEXT,
                cpf TEXT UNIQUE
            );
            CREATE TABLE IF NOT EXISTS contas (
                numero INTEGER PRIMARY KEY,
                cliente_id INTEGER,
                saldo REAL DEFAULT 0,
                FOREIGN KEY(cliente_id) REFERENCES clientes(id) ON DELETE CASCADE
            );
            CREATE TABLE IF NOT EXISTS extratos (
                id INTEGER PRIMARY KEY,
                numero_conta INTEGER,
                operacao TEXT,
                valor REAL,
                data TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY(numero_conta) REFERENCES contas(numero) ON DELETE CASCADE
            );
        """)
        self.conn.commit()

    def cpf_existe(self, cpf):
        return self.cursor.execute("SELECT 1 FROM clientes WHERE cpf = ?", (cpf,)).fetchone() is not None

    def inserir_cliente(self, nome, cpf):
        if self.cpf_existe(cpf):
            return None
        self.cursor.execute("INSERT INTO clientes (nome, cpf) VALUES (?, ?)", (nome, cpf))
        self.conn.commit()
        return self.cursor.lastrowid

    def inserir_conta(self, numero, cliente_id):
        self.cursor.execute("INSERT INTO contas (numero, cliente_id) VALUES (?, ?)", (numero, cliente_id))
        self.conn.commit()

    def atualizar_saldo(self, numero, saldo):
        self.cursor.execute("UPDATE contas SET saldo = ? WHERE numero = ?", (saldo, numero))
        self.conn.commit()

    def registrar_extrato(self, numero, op, valor):
        self.cursor.execute("INSERT INTO extratos (numero_conta, operacao, valor) VALUES (?, ?, ?)", (numero, op, valor))
        self.conn.commit()

    def buscar_conta_com_cliente(self, numero):
        return self.cursor.execute("""
            SELECT c.numero, c.saldo, cli.nome, cli.cpf FROM contas c
            JOIN clientes cli ON c.cliente_id = cli.id WHERE c.numero = ?
        """, (numero,)).fetchone()

    def obter_extrato(self, numero):
        return self.cursor.execute("""
            SELECT operacao, valor, data FROM extratos
            WHERE numero_conta = ? ORDER BY data
        """, (numero,)).fetchall()

    def excluir_cliente_por_cpf(self, cpf):
        c = self.cursor
        cliente = c.execute("SELECT id FROM clientes WHERE cpf = ?", (cpf,)).fetchone()
        if not cliente:
            return False
        cliente_id = cliente[0]
        contas = c.execute("SELECT numero FROM contas WHERE cliente_id = ?", (cliente_id,)).fetchall()
        for (num,) in contas:
            c.execute("DELETE FROM extratos WHERE numero_conta = ?", (num,))
        c.execute("DELETE FROM contas WHERE cliente_id = ?", (cliente_id,))
        c.execute("DELETE FROM clientes WHERE id = ?", (cliente_id,))
        self.conn.commit()
        return True


# Conta
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
        self.__extrato = extrato or []

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
    def saldo(self, v):
        self.__saldo = v

    @property
    def extrato(self):
        return self.__extrato

    @extrato.setter
    def extrato(self, v):
        self.__extrato = v

    def formatar_extrato(self):
        return "\n".join(f"{d[:16]} - {op}: R$ {v:.2f}" for op, v, d in self.__extrato) or "Nenhuma movimentação encontrada."


# Interface Gráfica
class BancoGUI:
    def __init__(self, root):
        self.root = root
        self.db = BancoDeDados()
        root.title("Banco Orion")
        root.geometry("500x500")
        root.configure(bg="#00276c")

        tk.Label(root, text="Banco Orion", font=("Arial", 20, "bold"), bg="#00276c", fg="#FFFFFF").pack(pady=20)

        botoes = [
            ("➕ Criar Conta", self.criar_cliente),
            ("💰 Depositar", self.depositar),
            ("💸 Sacar", self.sacar),
            ("🔄 Transferir", self.transferir),
            ("📋 Ver Extrato", self.ver_extrato),
            ("🗑️ Excluir Cliente", self.excluir_cliente)
        ]

        for txt, cmd in botoes:
            tk.Button(root, text=txt, command=cmd, font=("Arial", 12), width=35,
                      bg="#499AC0", fg="#FFFFFF", bd=0, pady=8, activebackground="#499AC0").pack(pady=5)

    def pedir_numero_conta(self, titulo="Conta", mensagem="Número da conta:"):
        n = simpledialog.askstring(titulo, mensagem)
        if n and n.isdigit():
            return int(n)
        else:
            messagebox.showerror("Erro", "Número inválido.")
            return None

    def pedir_cpf(self):
        cpf = simpledialog.askstring("CPF", "Digite o CPF do cliente:")
        if cpf:
            return cpf
        else:
            messagebox.showerror("Erro", "CPF inválido.")
            return None

    def buscar_conta(self, numero):
        if numero is None:
            return None
        dados = self.db.buscar_conta_com_cliente(numero)
        if dados:
            numero, saldo, nome, cpf = dados
            return Conta(numero, Cliente(nome, cpf), saldo, self.db.obter_extrato(numero))
        return None

    def criar_cliente(self):
        nome = simpledialog.askstring("Nome", "Nome do cliente:")
        cpf = simpledialog.askstring("CPF", "CPF do cliente:")
        num_str = simpledialog.askstring("Conta", "Número da nova conta:")

        if not (nome and cpf and num_str and num_str.isdigit()):
            messagebox.showerror("Erro", "Dados inválidos.")
            return

        if self.db.cpf_existe(cpf):
            messagebox.showerror("Erro", "CPF já cadastrado.")
            return

        cid = self.db.inserir_cliente(nome, cpf)
        if cid is None:
            messagebox.showerror("Erro", "Erro ao criar cliente.")
            return

        try:
            self.db.inserir_conta(int(num_str), cid)
        except sqlite3.IntegrityError:
            messagebox.showerror("Erro", "Número da conta já existe.")
            return

        messagebox.showinfo("Sucesso", f"Cliente {nome} cadastrado com a conta {num_str}.")

    def depositar(self):
        num = self.pedir_numero_conta()
        if num is None:
            return
        conta = self.buscar_conta(num)
        if not conta:
            messagebox.showerror("Erro", "Conta não encontrada.")
            return

        val_str = simpledialog.askstring("Depósito", "Valor a depositar:")
        try:
            valor = float(val_str)
            assert valor > 0
        except:
            messagebox.showerror("Erro", "Valor inválido!")
            return

        conta.saldo += valor
        self.db.atualizar_saldo(num, conta.saldo)
        self.db.registrar_extrato(num, "Depósito", valor)
        messagebox.showinfo("Sucesso", f"Depósito de R$ {valor:.2f} realizado.")

    def sacar(self):
        num = self.pedir_numero_conta()
        if num is None:
            return
        conta = self.buscar_conta(num)
        if not conta:
            messagebox.showerror("Erro", "Conta não encontrada.")
            return

        val_str = simpledialog.askstring("Saque", "Valor a sacar:")
        try:
            valor = float(val_str)
            assert 0 < valor <= conta.saldo
        except:
            messagebox.showerror("Erro", "Saldo insuficiente ou valor inválido!")
            return

        conta.saldo -= valor
        self.db.atualizar_saldo(num, conta.saldo)
        self.db.registrar_extrato(num, "Saque", valor)
        messagebox.showinfo("Sucesso", f"Saque de R$ {valor:.2f} realizado.")

    def transferir(self):
        origem = self.pedir_numero_conta("Transferência", "Número da conta de origem:")
        if origem is None:
            return
        destino = self.pedir_numero_conta("Transferência", "Número da conta de destino:")
        if destino is None:
            return

        if origem == destino:
            messagebox.showerror("Erro", "Conta de origem e destino não podem ser iguais.")
            return

        conta_origem = self.buscar_conta(origem)
        conta_destino = self.buscar_conta(destino)

        if not conta_origem or not conta_destino:
            messagebox.showerror("Erro", "Conta de origem ou destino não encontrada.")
            return

        val_str = simpledialog.askstring("Transferência", "Valor a transferir:")
        try:
            valor = float(val_str)
            assert 0 < valor <= conta_origem.saldo
        except:
            messagebox.showerror("Erro", "Saldo insuficiente!")
            return

        conta_origem.saldo -= valor
        conta_destino.saldo += valor

        self.db.atualizar_saldo(origem, conta_origem.saldo)
        self.db.atualizar_saldo(destino, conta_destino.saldo)

        self.db.registrar_extrato(origem, f"Transferência para {destino}", valor)
        self.db.registrar_extrato(destino, f"Transferência de {origem}", valor)

        messagebox.showinfo("Sucesso", f"Transferência de R$ {valor:.2f} realizada com sucesso.")

    def ver_extrato(self):
        num = self.pedir_numero_conta()
        if num is None:
            return
        conta = self.buscar_conta(num)
        if conta:
            messagebox.showinfo("Extrato", f"{conta.formatar_extrato()}\n\nSaldo atual: R$ {conta.saldo:.2f}")
        else:
            messagebox.showerror("Erro", "Conta não encontrada.")

    def excluir_cliente(self):
        cpf = self.pedir_cpf()
        if not cpf or not self.db.cpf_existe(cpf):
            messagebox.showerror("Erro", "CPF não cadastrado.")
            return

        if messagebox.askyesno("Confirmação", f"Deseja excluir o cliente com CPF {cpf} e suas contas?"):
            msg = "Cliente e contas excluídos." if self.db.excluir_cliente_por_cpf(cpf) else "Erro ao excluir cliente."
            messagebox.showinfo("Sucesso" if "excluídos" in msg else "Erro", msg)


# Execução
if __name__ == "__main__":
    root = tk.Tk()
    BancoGUI(root)
    root.mainloop()
