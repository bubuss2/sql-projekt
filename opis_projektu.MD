# Baza Danych Szkoły
Autorzy: Filip Szymeczko, Adrian Kacperski, Mikołaj Janusz

Projekt "Baza Danych Szkoły"

# Założenia projektu

Celem projektu jest stworzenie bazy danych szkoły ułatwiającej zarządzanie placówką poprzez m.in. monitorowanie wydatków, zarządzanie urlopami pracowników, w tym nauczycieli oraz wiele innych przydatnych funkcji.

Podstawowymi założeniami projektu była uniwersalność, ograniczyliśmy się do tabel występujących w większości szkół, więc w przypadku szkół np. specjalistycznych które mają unikalny system klas baza będzie ograniczona. Jednak baza może zostać łatwo rozwinięta o dodatkowe funkcjonalności. Również dane przykładowe są ograniczone, w praktyce szkoły mogą mieć znacznie więcej ocen, uczniów. Ograniliczyliśmy te dane aby skupić się na łatwym pokazaniu funkcjonalności.

## Schemat Pielęgnacji Bazy Danych

Jako że szkoła jest ważną instytucją publiczną i nie może sobie pozwolić na utracenie danych takich jak np. oceny, zalecane jest utworzenie różnicowej kopii zapasowej codziennie, w godzinnach nocnych gdy szkoła jest zamknięta i nie odbywają się zajęcia. Również zalecamy tworzenie cotygodniowej pełnej kopii zapasowej w nocnych godzinach weekendowych również z powodu małego prawdopodobieństwa używania bazy w tym czasie.

# Diagram ER
Na diagramie znajduję sie graficzna reprezentacja zależności pomiędzy tabelami w naszej bazie.
![DiagramER](diagram_ER.png)

# Schemat Bazy Danych
W schemacie bazy danych znajdują się wszystkie tabele i ich dane, typy tych danych, oraz dopuszczanie braku danej (NOT NULL).
![SchematBazy](schemat_bazy_danych.png)
# Tabele

- Administracja
- Członkowie kół
- Dni wolne
- Godziny lekcyjne
- Inni pracownicy
- Kierunki
- Klasy
- Kola naukowe
- Konkursy
- Nauczyciele
- Oceny
- Osoby
- Pracownicy
- Przedmioty
- Sale
- Sprzęt
- Stypendia
- Uczniowe
- Urlopy
- Wspolprace
- Wydatki
- Wynajem sal
- Wyniki konkursow
- Zajecia
- Zarzady klas
- Zastepstwa

# Widoki
Utworzone widoki pozwalają na pokazanie najczęściej według nas wybieranych danych. Nazwy widoków utworzone są tak aby było jasne co one wyświetlają	

```sql
CREATE VIEW [Spis klas] AS
	SELECT [Nazwa klasy], Nazwisko, Imie FROM Osoby
	JOIN Uczniowie ON Osoby.Id = Uczniowie.Id
	JOIN Klasy ON Uczniowie.Id_Klasy = Klasy.Id
	ORDER BY [Nazwa klasy], Nazwisko, Imie OFFSET 0 ROWS
```
```sql
CREATE VIEW [Najlepsi Stypendysci] AS
    SELECT Id_ucznia, SUM([Wysokosc stypendium]) AS [Wysokosc Stypendium] FROM Stypendia 
    GROUP BY Id_ucznia ORDER BY [Wysokosc stypendium] DESC OFFSET 0 ROWS
```
```sql
CREATE VIEW [Liczebnosc klas] AS
    SELECT [Nazwa klasy], COUNT(*) AS [Liczba osob] FROM Uczniowie
    JOIN Klasy ON Uczniowie.Id_Klasy = Klasy.Id
    GROUP BY [Nazwa klasy]
    ORDER BY [Nazwa klasy] OFFSET 0 ROWS
```

# Procedury Składowane
Procedury składowane w naszej bazie pozwalają np. na wypisanie uczniów na zadaną przez użytkownika literę, czy wypisanie średniej ocen jakieś określonej klasy.
```sql
GO
CREATE PROC uczniowie_na_litere
(@litera CHAR)
AS
SELECT Imie, Nazwisko FROM Uczniowie
JOIN Osoby ON Uczniowie.Id = Osoby.Id
WHERE Nazwisko LIKE @litera + '%'
ORDER BY Imie
GO
```
```sql
GO
CREATE PROC srednia_w_klasie
@klasa NVARCHAR(6)
AS
SELECT AVG(Ocena) AS Srednia FROM Oceny
JOIN Uczniowie ON Oceny.Id_ucznia = Uczniowie.Id
JOIN Klasy ON Uczniowie.Id_Klasy = Klasy.Id
WHERE [Nazwa klasy] = @klasa
GO
```
```sql
GO
CREATE PROC srednia_ucznia
@imie NVARCHAR(50),
@nazwisko NVARCHAR(50)
AS
SELECT AVG(Ocena) AS Srednia FROM Oceny
JOIN Uczniowie ON Oceny.Id_ucznia = Uczniowie.Id
JOIN Osoby ON Uczniowie.Id = Osoby.Id
WHERE Imie = @imie AND Nazwisko = @nazwisko
GO
```
# Wyzwalacze
Wyzwalacze w bazie są bardzo proste, np. dodawanie stypendium powoduje automatyczne zwiększenie kwoty wydatków na stypendia w tabeli wydatków, tak aby zautomatyzować aktualizację innych tabel. Lub aktualizowanie nazw sal czy klas według określonego schematu.

```sql
GO
CREATE TRIGGER nazwy_klas
ON Klasy
AFTER INSERT, UPDATE
AS
	UPDATE Klasy SET [Nazwa klasy] = CONCAT(Rok, [Skrot kierunku], Kod)

GO
```
```sql
GO
CREATE TRIGGER nazwy_sal
ON Sale
AFTER INSERT, UPDATE
AS
	UPDATE Sale SET [Nazwa sali] = CONCAT(Pietro, [Numer sali])
GO
```
```sql
GO
CREATE TRIGGER Dodaj_Wydatek_Stypendiow
ON Stypendia
AFTER INSERT
AS
    UPDATE Wydatki SET Koszt = (Koszt + (SELECT SUM(Stypendia.[Wysokosc stypendium]) FROM inserted, Stypendia WHERE inserted.Id = Stypendia.Id))
    WHERE Nazwa IN ('Stypendia')
GO
```
```sql
GO
CREATE TRIGGER Odejmij_Wydatek_Stypendium
ON Stypendia
INSTEAD OF DELETE
AS
    UPDATE Wydatki SET Koszt = (Koszt - (SELECT SUM(Stypendia.[Wysokosc stypendium]) FROM deleted, Stypendia WHERE deleted.Id = Stypendia.Id))
    WHERE Nazwa IN ('Stypendia')
    DELETE FROM Stypendia WHERE Stypendia.Id IN (SELECT Id FROM deleted)
GO
```
```sql
GO
CREATE TRIGGER Dodaj_Wydatek_Pracownik
ON Pracownicy
AFTER INSERT
AS
	UPDATE Wydatki SET Koszt = (Koszt + (SELECT SUM(Pracownicy.Pensja) FROM inserted, Pracownicy WHERE inserted.Id = Pracownicy.Id))
	WHERE Nazwa IN ('Pensje Pracownikow')
GO
```

# Skrypt Tworzący Bazę Danych
Skrypt tworzący baze tworzy wyżej wymienione tabelę, wyzwalacze, procedury składowane oraz widoki, upewniając się czy baza danych nie została już wczesniej utworzona.
Następnie wstawiane są przykładowe rekordy, które oczywiście można usunąć gdyż są one opcjonalne.
Pominięto na końcu tworzenie widoków, wyzwalaczy i procedur składowanych, gdyż jest to po prostu ten sam kod który został pokazany wcześniej. w przypadku opisów wyżej wymienionych punktów.

```sql
-- Usun poprzednia baze jesli istnieje o tej nazwie
IF EXISTS(select * from sys.databases where name='SzkolaDatabase')
	DROP DATABASE SzkolaDatabase
-- Utwórz bazę danych szkoły
CREATE DATABASE SzkolaDatabase
```

```sql
-- tworzenie tabel
USE SzkolaDatabase

CREATE TABLE Kierunki (
	Nazwa NVARCHAR(50) NOT NULL UNIQUE,
	Skrot NVARCHAR(3) PRIMARY KEY,
)

CREATE TABLE Klasy (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Rok INT NOT NULL,
	Kod NVARCHAR(3),
	[Skrot kierunku] NVARCHAR(3) REFERENCES Kierunki (Skrot) ON DELETE CASCADE ON UPDATE CASCADE,
	[Nazwa klasy] NVARCHAR(6),
)

CREATE TABLE Przedmioty (
	Nazwa NVARCHAR(50) PRIMARY KEY,
)

CREATE TABLE Osoby (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Imie NVARCHAR(50) NOT NULL,
	Nazwisko NVARCHAR(50) NOT NULL,
	Plec NVARCHAR(1) NOT NULL,
	[Data urodzenia] DATE NOT NULL,
)

CREATE TABLE Uczniowie (
	Id INT REFERENCES Osoby PRIMARY KEY,
	Id_Klasy INT REFERENCES Klasy ON DELETE CASCADE ON UPDATE CASCADE NOT NULL,
	[Numer telefonu opiekuna] NVARCHAR(10)
)

CREATE TABLE [Zarzad klas] (
	Id_ucznia INT REFERENCES Uczniowie PRIMARY KEY,
	Funkcja NVARCHAR(100) NOT NULL,
)

CREATE TABLE Pracownicy (
	Id INT REFERENCES Osoby PRIMARY KEY,
	Pensja MONEY NOT NULL,
	Etat FLOAT NOT NULL,
	[Numer telefonu] NVARCHAR(10)
)

CREATE TABLE Nauczyciele (
	Id INT REFERENCES Pracownicy PRIMARY KEY,
	Przedmiot NVARCHAR(50) REFERENCES Przedmioty (Nazwa),
)

CREATE TABLE [Inni pracownicy] (
	Id INT REFERENCES Pracownicy PRIMARY KEY,
	Profesja NVARCHAR(50) NOT NULL,
)

CREATE TABLE Administracja (
	Id INT REFERENCES Pracownicy PRIMARY KEY,
	[Nazwa stanowiska] NVARCHAR(50) NOT NULL,
	[Numer wewnetrzny] INT,
	[Adres email] NVARCHAR(100),
)

CREATE TABLE Oceny (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Id_ucznia INT REFERENCES Uczniowie NOT NULL,
	Id_nauczyciela INT REFERENCES Nauczyciele NOT NULL,
	Ocena FLOAT,
	Komentarz NVARCHAR(3000),
	Data DATE,
)

CREATE TABLE Sale (
	[Numer sali] NVARCHAR(30),
	Pietro INT,
	PRIMARY KEY ([Numer sali], Pietro),
	Przedmiot NVARCHAR(50) REFERENCES Przedmioty,
	Pojemnosc INT NOT NULL,
	[Nazwa sali] NVARCHAR(32)
)

CREATE TABLE [Godziny lekcyjne] (
	Lp INT PRIMARY KEY ,
	[Godzina rozpoczecia] DATETIME NOT NULL,
	[Godzina zakonczenia] DATETIME NOT NULL,
)

CREATE TABLE Zajecia (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Dzien NVARCHAR(20) NOT NULL,
	[Godzina lekcyjna] INT REFERENCES [Godziny lekcyjne],
	Klasa INT REFERENCES Klasy,
	Nauczyciel INT REFERENCES Nauczyciele NOT NULL,
	[Numer sali] NVARCHAR(30),
	Pietro INT,
	FOREIGN KEY ([Numer sali], Pietro) REFERENCES Sale ([Numer sali], Pietro),
)

CREATE TABLE [Kola naukowe] (
	Nazwa NVARCHAR(100) PRIMARY KEY,
	[Liczba czlonkow] INT,
	[Rok zalozenia] INT,
	Opiekun INT REFERENCES Nauczyciele,
)

CREATE TABLE [Czlonkowie kol] (
	Id_ucznia INT REFERENCES Uczniowie,
	Nazwa_kola NVARCHAR(100) REFERENCES [Kola naukowe],
	Aktywny BIT NOT NULL,
	PRIMARY KEY (Id_ucznia, Nazwa_kola),
)

CREATE TABLE Wspolpraca (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	[Nazwa podmiotu] NVARCHAR(100),
	Adres NVARCHAR(1000),
)

CREATE TABLE Urlopy (
	Id_pracownika INT REFERENCES Pracownicy PRIMARY KEY,
	[Data rozpoczecia] DATE NOT NULL,
	[Data zakonczenia] DATE NOT NULL,
)

CREATE TABLE Zastepstwa (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Id_zajec INT REFERENCES Zajecia,
	Id_nauczyciela INT REFERENCES Nauczyciele,
	[Data rozpoczecia] DATE NOT NULL,
	[Data zakonczenia] DATE NOT NULL,
)

CREATE TABLE [Dni wolne] (
	Dzien DATE PRIMARY KEY,
)

CREATE TABLE [Wynajem sal] (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	[Godzina rozpoczecia] DATETIME,
	[Godzina zakonczenia] DATETIME,
	Koszt MONEY,
	[Numer sali] NVARCHAR(30),
	Pietro INT,
	FOREIGN KEY ([Numer sali], Pietro) REFERENCES Sale ([Numer sali], Pietro),
)

CREATE TABLE Konkursy (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Nazwa NVARCHAR(100),
	Opiekun INT REFERENCES Nauczyciele,
)

CREATE TABLE [Wyniki konkursow] (
	Id_konkursu INT REFERENCES Konkursy,
	Id_ucznia INT REFERENCES Uczniowie,
	Miejsce INT,
	PRIMARY KEY (Id_konkursu, Id_ucznia),
)

CREATE TABLE Sprzet (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Nazwa NVARCHAR(100),
	[Liczba sztuk] INT,
	[Numer sali] NVARCHAR(30),
	Pietro INT,
	FOREIGN KEY ([Numer sali], Pietro) REFERENCES Sale ([Numer sali], Pietro),
)

CREATE TABLE Wydatki (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	Nazwa NVARCHAR(100),
	Koszt MONEY,
)

CREATE TABLE Stypendia (
	Id INT IDENTITY(1,1) PRIMARY KEY,
	[Nazwa stypendium] NVARCHAR(100),
	Id_ucznia INT REFERENCES Uczniowie,
	[Wysokosc stypendium] MONEY,
)
```

### Przykładowe Dane do bazy danych
```sql
-- wstawianie wartosci
INSERT INTO Kierunki VALUES
('mat-inf', 'MI'),
('bio-chem', 'BC'),
('bio-geo', 'BG'),
('mat-his', 'MH'),
('human', 'HM'),
('mat-fiz','MF')

INSERT INTO Przedmioty VALUES
('matematyka'),
('język polski'),
('język angielski'),
('chemia'),
('fizyka'),
('biologia')

INSERT INTO Klasy(Rok, Kod, [Skrot kierunku]) VALUES
(1, 'a', 'MF'),
(1, 'b', 'MH'),
(2, 'a', 'MI'),
(2, 'b', 'BC'),
(3, 'b', 'HM'),
(3, 'a', 'BC')

INSERT INTO Osoby VALUES
('Jan', 'Kowalski', 'M', '2007-07-15'),
('Karol', 'Matyla', 'M', '2006-04-02'),
('Wojciech', 'Wotyla', 'M', '2008-05-18'),
('Andżelina', 'Dżoli', 'K', '2007-10-24'),
('Oliwia', 'Brazylia', 'K', '1995-12-03'),
('Cygan', 'Dziewięć', 'M', '2006-06-06'),
('Bogumiła', 'Mazur', 'K', '1987-11-13'),
('Ernest', 'Chlebek', 'M', '1989-11-13'),
('Rafał', 'Warszawski', 'M', '2009-11-13'),
('Wiktor', 'Mały', 'M', '2010-01-11'),
('Eustachy', 'Szewc', 'M', '1956-01-11'),
('Stefan', 'Filipowski', 'M', '1984-02-18'),
('Liwia', 'Wasilewski', 'K', '2008-10-18'),
('Zygfryd', 'Jaskólski', 'M', '2011-06-26'),
('Kamila', 'Kaczmarek', 'K', '2009-03-31'),
('Elżbieta', 'Wójcik', 'K', '1967-03-31'),
('Monika', 'Pokorny', 'K', '2007-11-07'),
('Ferdynand', 'Brzezicki', 'M', '1985-12-24'),
('Malgorzata', 'Fabian', 'K', '1965-10-12'),
('Szczęsny', 'Chmiel', 'M', '1944-04-01'),
('Wiktoria' , 'Sowa' , 'K' , '1992-05-10')

INSERT INTO Uczniowie VALUES
(1, 1, '123456789'),
(2, 2, '234567891'),
(3, 3, '345678912'),
(4, 4, '456789123'),
(6, 5, '883623477'),
(9, 6, '883623477'),
(10, 1, '883683478'),
(13, 2, '123123123'),
(14, 3, '123123123'),
(12, 3, '123123123'),
(16, 6, '123423153')

INSERT INTO [Zarzad Klas] VALUES
(1, 'Przewodniczacy'),
(4, 'Vice-Przewodniczacy'),
(6, 'Przewodniczacy'),
(9, 'Skarbnik'),
(12, 'Przewodniczacy'),
(16, 'Vice-Przewodniczacy')

INSERT INTO Pracownicy VALUES
(5, 4000, 1, '696969420'),
(7, 4000, 1, '420420420'),
(8, 4200, 1, '120120120'),
(11, 5600, 1, '3885690453'),
(15, 3500, 0.6, '123498765'),
(17, 4500, 0.8, '123498765'),
(18, 4000, 1, '123498765'),
(19, 3500, 1, '123498765'),
(20, 4200, 1, '123498765'),
(21, 4600, 1, '129384756')

INSERT INTO Nauczyciele VALUES
(5, 'matematyka'),
(8, 'język polski'),
(15, 'język angielski'),
(18, 'chemia'),
(19, 'fizyka'),
(7, 'biologia')

INSERT INTO [Inni pracownicy] VALUES
(17, 'Kucharka'),
(20, 'Woźny')

INSERT INTO Administracja VALUES
(11, 'Dyrektor', '48', 'koxdyro123@onet.pl'),
(21, 'Sekretarka', '48', 'dydaktyka.wmier@szkola.pl')

INSERT INTO Sale([Numer sali], Pietro, Przedmiot, Pojemnosc) VALUES
(21, 1, 'matematyka', 32),
(22, 1, 'język polski', 30),
(23, 1, 'fizyka', 32),
(21, 2, 'biologia', 16),
(22, 2, NULL, 16),
(23, 2, 'chemia', 26),
(24, 2, 'język angielski', 18)

INSERT INTO Sprzet (Nazwa, [Liczba sztuk], [Numer sali], Pietro) VALUES
('Przyrzady miernicze', 5, 21, 1),
('Przyrzady miernicze', 4, 23, 1),
('Przyrzady miernicze', 7, 23, 2),
('Szkielet człowieka', 1, 21, 2),
('Słownik językowy' , 15, 22, 1),
('Słownik językowy' , 10, 24, 2)

INSERT INTO [Kola naukowe](Nazwa, [Rok zalozenia], Opiekun) VALUES
('Koło Fanów Kołomogorowa' , 2021, 5),
('Koło Fizyków' , 2019, 19),
('Koło Uczniów Informatyki' , 2009, 15)

INSERT INTO [Czlonkowie kol](Id_ucznia, Nazwa_kola, Aktywny) VALUES 
(2, 'Koło Fanów Kołomogorowa', 1),
(2, 'Koło Uczniów Informatyki', 1),
(6, 'Koło Fizyków', 1),
(12, 'Koło Fizyków', 0),
(13, 'Koło Uczniów Informatyki', 1),
(14, 'Koło Fanów Kołomogorowa', 1),
(14, 'Koło Uczniów Informatyki', 0),
(16, 'Koło Fizyków', 1)

INSERT INTO Wspolpraca VALUES
(N'Szkola Podstawowa w Jarzynowie Dolnym', N'Ulica Jarzynowa 3H, Polska'),
(N'Harvard University', N'Cambridge, MA, Stany Zjednoczone'),
(N'Ambasada Australii w Warszawie', N'Nowogrodzka 11, Warszawa'),
(N'Google Poland', N'Plac Konsera 10, Warszawa')

INSERT INTO [Dni wolne] VALUES
('2022-01-01'),
('2022-01-06'),
('2022-04-13'),
('2022-05-01'),
('2022-06-11'),
('2022-08-15'),
('2022-11-11'),
('2022-12-25'),
('2022-12-26')

INSERT INTO Wydatki VALUES
('Pensje Pracownikow', 478212.65),
('Naprawa uszkodzeń toalety', 3200),
('Przybory Szkolne', 12600),
('Sprzęt Sportowy', 16320),
('Stypendia', 35000),
('Organizacja Wydarzeń', 67100),
('Koks', 1700),
('Utrzymanie budynku szkoły', 163000)

INSERT INTO [Wynajem sal] VALUES
('2022-02-09 18:30', '2022-09-02 20:30', 300.5, 21, 1),
('2022-01-30 17:00', '2022-09-02 20:30', 300.5, 22, 2),
('2022-09-02 18:30', '2022-09-02 20:30', 300.5, 21, 2),
('2022-09-02 18:30', '2022-09-02 20:30', 300.5, 21, 1),
('2022-09-02 18:30', '2022-09-02 20:30', 300.5, 21, 1)

INSERT INTO Konkursy(Nazwa, Opiekun) VALUES
('Konkurs jedzenia brokuła na czas', 5),
('Olimpiada Matematyczna dla biedaków', 5),
('Dyktando powiatowe', 8),
('Spiewanie Koled z woznym any%', 15)

INSERT INTO [Wyniki konkursow] VALUES
(1, 2, 3),
(1, 16, 2),
(1, 12, 1),
(2, 1, 3),
(2, 4, 2),
(2, 9, 1),
(3, 13, 3),
(3, 2, 2),
(3, 12, 1),
(4, 3, 3),
(4, 10, 2),
(4, 4, 1)

INSERT INTO Stypendia VALUES
('Medal Honoru', 2, 7500),
('Paszport Polsatu', 3, 500),
('Order Uśmiechu', 3, 10),
('Stypendium Dziekana Uniwersytetu w Jarzynowie', 2, 2500)

INSERT INTO Urlopy VALUES
(20, '2022-01-07', '2022-01-16'),
(5, '2022-02-09', '2022-02-28')

INSERT INTO Oceny(Id_ucznia, Id_nauczyciela, Ocena, Data) VALUES
(1, 5, 4, '2022-01-12'),
(2, 5, 2, '2022-01-12'),
(3, 5, 5, '2022-01-12'),
(4, 5, 3.5, '2022-01-12'),
(6, 5, 3, '2022-01-12'),
(9, 5, 4, '2022-01-12'),
(10, 5, 2.75, '2022-01-12'),
(12, 5, 4.5, '2022-01-12'),
(13, 5, 2, '2022-01-12'),
(1,8, 3, '2021-12-17'),
(16,8, 3, '2021-12-17'),
(14,8, 2, '2021-12-17'),
(13,8, 1, '2021-12-17'),
(12,8, 3.5, '2021-12-17'),
(1,8, 2, '2021-12-17'),
(10,8, 4.75, '2021-12-17'),
(6,8, 4, '2021-12-17'),
(4,8, 2, '2021-12-17'),
(2,8, 6, '2021-12-17'),
(1,18, 5, '2021-12-10'),
(2,18, 5, '2021-12-10'),
(3,18, 5, '2021-12-10'),
(4,18, 4, '2021-12-10'),
(6,18, 4.5, '2021-12-10'),
(9,18, 5, '2021-12-10'),
(10,18, 5, '2021-12-10'),
(1,18, 4, '2021-12-10'),
(12,18, 3.75, '2021-12-10'),
(13,18, 4, '2021-12-10'),
(14,18, 5, '2021-12-10'),
(16,18, 4.75, '2021-12-10')

INSERT INTO [Godziny lekcyjne] VALUES
(1, '08:00', '08:45'),
(2, '09:00', '09:45'),
(3, '10:00', '10:45'),
(4, '11:00', '11:45'),
(5, '12:00', '12:45'),
(6, '13:00', '13:45')

INSERT INTO Zajecia(Dzien,[Godzina Lekcyjna],Klasa,Nauczyciel,[Numer sali], Pietro) VALUES
('Poniedzialek', 1, 1, 5, 21, 1),
('Poniedzialek', 2, 1, 18, 23, 2),
('Poniedzialek', 3, 1, 19, 23, 1),
('Poniedzialek', 4, 1, 15, 22, 2),
('Poniedzialek', 5, 1, 7, 21, 2),
('Poniedzialek', 1, 2, 8, 22, 1),
('Poniedzialek', 2, 2, 5, 21, 1),
('Poniedzialek', 3, 2, 15, 24, 2),
('Poniedzialek', 4, 2, 19, 23, 1),
('Poniedzialek', 5, 2, 18, 23, 2),
('Poniedzialek', 1, 3, 15, 24, 2),
('Poniedzialek', 2, 3, 15, 24, 2),
('Poniedzialek', 3, 3, 5, 21, 1),
('Poniedzialek', 4, 3, 5, 21, 1),
('Poniedzialek', 5, 3, 8, 22, 1),
('Poniedzialek', 1, 4, 18, 21, 1),
('Poniedzialek', 2, 4, 7, 21, 2),
('Poniedzialek', 3, 4, 8, 22, 1),
('Poniedzialek', 4, 4, 8, 22, 1),
('Poniedzialek', 5, 4, 19, 23, 1),
('Wtorek', 1, 1, 8, 22, 1),
('Wtorek', 2, 1, 5, 21, 1),
('Wtorek', 3, 1, 15, 22, 2),
('Wtorek', 4, 1, 18, 23, 2),
('Wtorek', 5, 1, 7, 21, 2),
('Wtorek', 1, 2, 18, 23, 2),
('Wtorek', 2, 2, 8, 22, 1),
('Wtorek', 3, 2, 5, 21, 1),
('Wtorek', 4, 2, 15, 24, 2),
('Wtorek', 5, 2, 19, 23, 1),
('Wtorek', 6, 2, 7, 21, 2),
('Wtorek', 1, 3, 7, 21, 2),
('Wtorek', 2, 3, 19, 23, 1),
('Wtorek', 3, 3, 8, 22, 1),
('Wtorek', 4, 3, 5, 21, 1),
('Wtorek', 5, 3, 18, 23, 2),
('Wtorek', 1, 4, 15, 24, 2),
('Wtorek', 2, 4, 15, 24, 2),
('Wtorek', 3, 4, 19, 23, 1),
('Wtorek', 4, 4, 7, 21, 2),
('Wtorek', 5, 4, 5, 21, 1),
('Wtorek', 6, 4, 8, 22, 1),
('Środa', 1, 1, 8, 22, 1),
('Środa', 2, 1, 5, 21, 1),
('Środa', 3, 1, 5, 21, 1),
('Środa', 4, 1, 19, 23, 1),
('Środa', 1, 2, 15, 24, 2),
('Środa', 2, 2, 15, 24, 2),
('Środa', 3, 2, 7, 21, 2),
('Środa', 1, 3, 5, 21, 1),
('Środa', 2, 3, 8, 22, 1),
('Środa', 3, 3, 19, 23, 1),
('Środa', 4, 3, 7, 21, 2),
('Środa', 5, 3, 18, 23, 2),
('Środa', 6, 3, 15, 24, 2),
('Środa', 1, 4, 19, 23, 1),
('Środa', 2, 4, 7, 21, 2),
('Środa', 3, 4, 15, 22, 2),
('Środa', 4, 4, 5, 21, 1),
('Środa', 5, 4, 5, 21, 1),
('Czwartek', 1, 1, 15, 24, 2),
('Czwartek', 2, 1, 15, 24, 2),
('Czwartek', 3, 1, 19, 23, 1),
('Czwartek', 4, 1, 7, 21, 2),
('Czwartek', 5, 1, 8, 22, 1),
('Czwartek', 1, 2, 5, 21, 1),
('Czwartek', 2, 2, 5, 21, 1),
('Czwartek', 3, 2, 8, 22, 1),
('Czwartek', 4, 2, 19, 23, 1),
('Czwartek', 5, 2, 7, 21, 2),
('Czwartek', 1, 3, 7, 21, 2),
('Czwartek', 2, 3, 19, 23, 1),
('Czwartek', 3, 3, 15, 24, 2),
('Czwartek', 3, 4, 5, 21, 1),
('Czwartek', 4, 4, 15, 22, 2),
('Czwartek', 5, 4, 18, 23, 2)

INSERT INTO Zastepstwa(Id_zajec, Id_nauczyciela, [Data rozpoczecia], [Data zakonczenia]) VALUES
(1, 19, '2022-01-23', '2022-02-27'),
(31, 5, '2021-12-10', '2022-01-04')
```


## Przykładowe Zapytania
Przykładowymi zapytaniami mogą być dowolne pytania jakie możemy mieć o aktualnej sytuacji szkoły, zapytania o oceny danego ucznia, tabeli wydatków czy zwolnienia danego nauczyciela.