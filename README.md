# 👥 Mini Projeto SQL - Sistema de RH

Este projeto tem como objetivo simular um **Sistema de Recursos Humanos
(RH)** utilizando **MySQL**, com modelagem de dados, inserção de
informações fictícias, consultas analíticas e uso de **Views, Functions
e Procedures**.

O projeto foca em praticar **modelagem de banco de dados relacional** e
**análises de informações de colaboradores, presenças e folha de
pagamento**.

------------------------------------------------------------------------

## 🗂 Criação do Banco de Dados

``` sql
DROP DATABASE IF EXISTS SistemaRH;
CREATE DATABASE SistemaRH;
USE SistemaRH;
```

------------------------------------------------------------------------

## 🏗 Estrutura das Tabelas

### 1. Tabela `Departamentos`

``` sql
CREATE TABLE Departamentos (
  DepartamentoID INT PRIMARY KEY AUTO_INCREMENT,
  Nome VARCHAR(100) NOT NULL
);
```

### 2. Tabela `Cargos`

``` sql
CREATE TABLE Cargos (
  CargoID INT PRIMARY KEY AUTO_INCREMENT,
  Nome VARCHAR(100) NOT NULL,
  SalarioBase DECIMAL(10,2) NOT NULL
);
```

### 3. Tabela `Funcionarios`

``` sql
CREATE TABLE Funcionarios (
  FuncionarioID INT PRIMARY KEY AUTO_INCREMENT,
  Nome VARCHAR(100) NOT NULL,
  Email VARCHAR(100) UNIQUE,
  DataAdmissao DATE DEFAULT CURDATE(),
  DepartamentoID INT,
  CargoID INT,
  FOREIGN KEY (DepartamentoID) REFERENCES Departamentos(DepartamentoID),
  FOREIGN KEY (CargoID) REFERENCES Cargos(CargoID)
);
```

### 4. Tabela `Presencas`

``` sql
CREATE TABLE Presencas (
  PresencaID INT PRIMARY KEY AUTO_INCREMENT,
  FuncionarioID INT,
  Data DATE NOT NULL,
  Status ENUM('Presente','Falta','Atraso') DEFAULT 'Presente',
  FOREIGN KEY (FuncionarioID) REFERENCES Funcionarios(FuncionarioID)
);
```

### 5. Tabela `FolhaPagamento`

``` sql
CREATE TABLE FolhaPagamento (
  FolhaID INT PRIMARY KEY AUTO_INCREMENT,
  FuncionarioID INT,
  MesReferencia VARCHAR(7), -- Ex: 2025-08
  SalarioBase DECIMAL(10,2),
  HorasExtras DECIMAL(10,2) DEFAULT 0,
  Descontos DECIMAL(10,2) DEFAULT 0,
  SalarioFinal DECIMAL(10,2),
  FOREIGN KEY (FuncionarioID) REFERENCES Funcionarios(FuncionarioID)
);
```

------------------------------------------------------------------------

## 📥 Inserindo Dados Fictícios

### Departamentos

``` sql
INSERT INTO Departamentos (Nome) VALUES
('Financeiro'), ('Recursos Humanos'), ('TI'), ('Vendas');
```

### Cargos

``` sql
INSERT INTO Cargos (Nome, SalarioBase) VALUES
('Analista Financeiro', 4500.00),
('Analista RH', 4000.00),
('Desenvolvedor', 6000.00),
('Vendedor', 3000.00);
```

### Funcionários

``` sql
INSERT INTO Funcionarios (Nome, Email, DepartamentoID, CargoID) VALUES
('João Silva', 'joao@empresa.com', 1, 1),
('Maria Souza', 'maria@empresa.com', 2, 2),
('Carlos Pereira', 'carlos@empresa.com', 3, 3),
('Ana Costa', 'ana@empresa.com', 4, 4);
```

### Presenças

``` sql
INSERT INTO Presencas (FuncionarioID, Data, Status) VALUES
(1, '2025-08-01', 'Presente'),
(1, '2025-08-02', 'Falta'),
(2, '2025-08-01', 'Presente'),
(3, '2025-08-01', 'Atraso'),
(4, '2025-08-01', 'Presente');
```

### Folha de Pagamento

``` sql
INSERT INTO FolhaPagamento (FuncionarioID, MesReferencia, SalarioBase, HorasExtras, Descontos, SalarioFinal) VALUES
(1, '2025-08', 4500.00, 200.00, 100.00, 4600.00),
(2, '2025-08', 4000.00, 150.00, 0.00, 4150.00),
(3, '2025-08', 6000.00, 300.00, 200.00, 6100.00),
(4, '2025-08', 3000.00, 100.00, 0.00, 3100.00);
```

------------------------------------------------------------------------

## 🔎 Consultas Analíticas

### Média salarial por departamento

``` sql
SELECT d.Nome AS Departamento, AVG(f.SalarioFinal) AS MediaSalarial
FROM FolhaPagamento f
JOIN Funcionarios fu ON f.FuncionarioID = fu.FuncionarioID
JOIN Departamentos d ON fu.DepartamentoID = d.DepartamentoID
GROUP BY d.Nome;
```

### Funcionários com mais horas extras

``` sql
SELECT fu.Nome, SUM(f.HorasExtras) AS TotalHorasExtras
FROM FolhaPagamento f
JOIN Funcionarios fu ON f.FuncionarioID = fu.FuncionarioID
GROUP BY fu.Nome
ORDER BY TotalHorasExtras DESC;
```

### Ranking de faltas

``` sql
SELECT fu.Nome, COUNT(*) AS TotalFaltas
FROM Presencas p
JOIN Funcionarios fu ON p.FuncionarioID = fu.FuncionarioID
WHERE p.Status = 'Falta'
GROUP BY fu.Nome
ORDER BY TotalFaltas DESC;
```

------------------------------------------------------------------------

## 🛠 Recursos Avançados

### VIEW: Relatório de Folha

``` sql
CREATE OR REPLACE VIEW RelatorioFolha AS
SELECT fu.Nome AS Funcionario,
       c.Nome AS Cargo,
       d.Nome AS Departamento,
       f.MesReferencia,
       f.SalarioBase,
       f.HorasExtras,
       f.Descontos,
       f.SalarioFinal
FROM FolhaPagamento f
JOIN Funcionarios fu ON f.FuncionarioID = fu.FuncionarioID
JOIN Cargos c ON fu.CargoID = c.CargoID
JOIN Departamentos d ON fu.DepartamentoID = d.DepartamentoID;
```

### FUNCTION: Cálculo de absenteísmo

``` sql
DELIMITER //
CREATE FUNCTION FaltasFuncionario(func INT) RETURNS INT
DETERMINISTIC
BEGIN
  DECLARE totalFaltas INT;
  SELECT COUNT(*) INTO totalFaltas
  FROM Presencas
  WHERE FuncionarioID = func AND Status = 'Falta';
  RETURN totalFaltas;
END//
DELIMITER ;
```

### PROCEDURE: Adicionar funcionário e folha inicial

``` sql
DELIMITER //
CREATE PROCEDURE NovoFuncionario(
  IN nome VARCHAR(100),
  IN email VARCHAR(100),
  IN depto INT,
  IN cargo INT
)
BEGIN
  DECLARE novoID INT;
  DECLARE salario DECIMAL(10,2);

  -- Insere funcionário
  INSERT INTO Funcionarios (Nome, Email, DepartamentoID, CargoID)
  VALUES (nome, email, depto, cargo);

  SET novoID = LAST_INSERT_ID();

  -- Busca salário base do cargo
  SELECT SalarioBase INTO salario FROM Cargos WHERE CargoID = cargo;

  -- Insere folha inicial
  INSERT INTO FolhaPagamento (FuncionarioID, MesReferencia, SalarioBase, HorasExtras, Descontos, SalarioFinal)
  VALUES (novoID, DATE_FORMAT(CURDATE(),'%Y-%m'), salario, 0, 0, salario);
END//
DELIMITER ;
```

------------------------------------------------------------------------

## 🚀 Tecnologias Utilizadas

-   **MySQL**
-   **SQL (DDL, DML, DQL)**
-   Views, Functions, Procedures
-   Modelagem de dados relacional aplicada a RH

------------------------------------------------------------------------

## 🎯 Objetivo do Projeto

-   Criar um modelo de banco de dados para **gestão de RH**\
-   Simular **presenças, salários e absenteísmo**\
-   Utilizar recursos de SQL para **análises de colaboradores e folha de
    pagamento**

------------------------------------------------------------------------

## 📖 Próximos Passos

-   Integrar os dados com **Power BI** para visualização de indicadores
    de RH\
-   Expandir para incluir **avaliações de desempenho** e **benefícios**\
-   Implementar **regras de negócio adicionais** (férias, promoções,
    reajustes)

------------------------------------------------------------------------

👤 **Autor:** [Frederico Feitosa
Funis](www.linkedin.com/in/frederico-funis-952123215)\
📧 **Contato:** disponível via LinkedIn
