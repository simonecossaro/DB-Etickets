CREATE DATABASE IF NOT EXISTS eTickets;

USE eTickets;

CREATE TABLE squadra(
     nome varchar(45),
     sport varchar(45) NOT NULL,
     sede varchar(45) NOT NULL,
     PRIMARY KEY (nome)
);

CREATE TABLE arena(
     nome varchar(45),
     via varchar(45) NOT NULL,
     numero varchar(8),
     città varchar(45) NOT NULL,
     capienza int(6) NOT NULL,
     tipo varchar(10) NOT NULL,
     PRIMARY KEY (nome),
     CONSTRAINT check_tip_arena CHECK (tipo = 'Aperto' OR tipo = 'Chiuso'),
     CONSTRAINT check_cap_arena CHECK (capienza > 0)
);

CREATE TABLE cantante(
     nome varchar(45),
     genere varchar(45) NOT NULL,
     nazionalità varchar(45) NOT NULL,
     PRIMARY KEY (nome)
);

CREATE TABLE spettatore(
    nome            varchar(45) NOT NULL,
    cognome         varchar(45) NOT NULL,
    codicefiscale   varchar(45),
    datanascita     date        NOT NULL,
    comuneresidenza varchar(45) NOT NULL,
    telefono        varchar(20) NOT NULL,
    email           varchar(45) NOT NULL,
    PRIMARY KEY (codicefiscale)
);

CREATE TABLE rivenditore(
     codice int(6) AUTO_INCREMENT,
     nome varchar(45) NOT NULL,
     via varchar(45) NOT NULL,
     numero varchar(8) NOT NULL,
     città varchar(45) NOT NULL,
     telefono varchar(20) NOT NULL,
     email varchar(45) NOT NULL,
     PRIMARY KEY (codice)
);

CREATE TABLE biglietto(
     id int(11) AUTO_INCREMENT,
     costo float(6,2) NOT NULL,
     disponibilità int(6) NOT NULL,
     PRIMARY KEY (id),
     CONSTRAINT check_costo_bigl CHECK (costo >= 0),
     CONSTRAINT check_disp_bigl CHECK (disponibilità >= 0)
);

CREATE TABLE partita(
     id int(11) AUTO_INCREMENT,
     data date NOT NULL,
     ora time NOT NULL,
     arena varchar(45) NOT NULL,
     competizione varchar(45) NOT NULL,
     locali varchar(45) NOT NULL,
     ospiti varchar(45) NOT NULL,
     biglietto int(11) UNIQUE,
     PRIMARY KEY (id),
     CONSTRAINT FK_arena_par FOREIGN KEY (arena) REFERENCES arena(nome),
     CONSTRAINT FK_loc FOREIGN KEY (locali) REFERENCES squadra(nome),
     CONSTRAINT FK_osp FOREIGN KEY (ospiti) REFERENCES squadra(nome),
     CONSTRAINT FK_bigl_par FOREIGN KEY (biglietto) REFERENCES biglietto(id) ON DELETE CASCADE
);

CREATE TABLE concerto(
     id int(11) AUTO_INCREMENT,
     data date NOT NULL,
     ora time NOT NULL,
     arena varchar(45) NOT NULL,
     artista varchar(45) NOT NULL,
     biglietto int(11) UNIQUE,
     PRIMARY KEY (id),
     CONSTRAINT FK_arena_conc FOREIGN KEY (arena) REFERENCES arena(nome),
     CONSTRAINT FK_artista FOREIGN KEY (artista) REFERENCES cantante(nome),
     CONSTRAINT FK_bigl_conc FOREIGN KEY (biglietto) REFERENCES biglietto(id) ON DELETE CASCADE
);

CREATE TABLE rivendita(
     id int(11) AUTO_INCREMENT,
     biglietto int(11) NOT NULL,
     rivenditore int(6) NOT NULL,
     datacessione datetime NOT NULL,
     PRIMARY KEY (id),
     CONSTRAINT FK_riv FOREIGN KEY (rivenditore) REFERENCES rivenditore(codice) ON DELETE CASCADE,
     CONSTRAINT FK_bigl_riv FOREIGN KEY (biglietto) REFERENCES biglietto(id) ON DELETE CASCADE
);

CREATE TABLE acquisto(
     id int(11) AUTO_INCREMENT,
     acquirente varchar(45) NOT NULL,
     biglietto int(11) NOT NULL,
     dataacquisto datetime NOT NULL,
     PRIMARY KEY (id),
     CONSTRAINT FK_acquirente FOREIGN KEY (acquirente) REFERENCES spettatore(codicefiscale),
     CONSTRAINT FK_bigl_acq FOREIGN KEY (biglietto) REFERENCES biglietto(id) ON DELETE CASCADE
);

DELIMITER $$
CREATE TRIGGER trg_bef_partita
BEFORE INSERT ON partita
FOR EACH ROW
BEGIN
  SET @disp = 0;
  SET @cap = 0;
  SET @s1 = '';
  SET @s2 = '';
  SET @c = 0;
  IF NEW.data < now() THEN signal sqlstate '45007'
     SET message_text = 'La data non può essere passata';
  END IF;
  SELECT disponibilità INTO @disp FROM biglietto WHERE id = NEW.biglietto;
  SELECT capienza INTO @cap FROM arena WHERE nome = NEW.arena;
  IF (@disp > @cap) THEN signal sqlstate '45011' SET message_text = 'Biglietto
                           con disponibilità maggiore alla capienza';
  END IF;
  IF (NEW.locali = NEW.ospiti) THEN signal sqlstate '45012'
     SET message_text = 'Una squadra non può giocare contro sè stessa';
  END IF;
  SELECT sport INTO @s1 FROM squadra WHERE nome = NEW.locali;
  SELECT sport INTO @s2 FROM squadra WHERE nome = NEW.ospiti;
  IF (@s1 != @s2) THEN signal sqlstate '45013'
     SET message_text = 'Squadre di sport diversi';
  END IF;
SELECT count(*) INTO @c FROM concerto
WHERE biglietto = NEW.biglietto
GROUP BY biglietto;
IF (@c != 0) THEN signal sqlstate '45027' SET message_text = 'Biglietto legato ad un altro evento';
  END IF;
END;$$
DELIMITER;

DELIMITER $$
CREATE TRIGGER trg_bef_concerto
BEFORE INSERT ON concerto
FOR EACH ROW
BEGIN
  SET @disp = 0;
  SET @cap = 0;
  SET @c = 0;
  IF NEW.data < now() THEN signal sqlstate '45007'
     SET message_text = 'La data non può essere passata';
  END IF;
  SELECT disponibilità INTO @disp FROM biglietto WHERE id = NEW.biglietto;
  SELECT capienza INTO @cap FROM arena WHERE nome = NEW.arena;
  IF (@disp > @cap) THEN signal sqlstate '45011' SET message_text = 'Biglietto
                           con disponibilità maggiore alla capienza';
  END IF;
  SELECT count(*) INTO @c FROM partita
  WHERE biglietto = NEW.biglietto
  GROUP BY biglietto;
  IF (@c != 0) THEN signal sqlstate '45027' SET message_text = 'Biglietto legato ad un altro evento';
  END IF;
END;$$
DELIMITER;

DELIMITER $$
CREATE TRIGGER trig_bef_acquisto
BEFORE INSERT ON acquisto
FOR EACH ROW
BEGIN
  SET @c = 0;
  SET @d = 0;
  SELECT disponibilità INTO @d FROM biglietto
       WHERE (biglietto.id = NEW.biglietto);
  SELECT count(*) INTO @c FROM acquisto
       WHERE ((acquirente = NEW.acquirente) AND (biglietto = NEW.biglietto));
  IF (@c != 0) THEN signal sqlstate '45008' SET message_text = 'Ha già
               acquistato questo biglietto';
  END IF;
  IF (@d > 0) THEN
       UPDATE biglietto
       SET disponibilità = disponibilità - 1
       WHERE id = NEW.biglietto;
  ELSE signal sqlstate '45010' SET message_text = 'Non disponibile';
  END IF;
END; $$
DELIMITER;

DELIMITER $$
CREATE TRIGGER trig_disp_riv
BEFORE INSERT ON rivendita
FOR EACH ROW
BEGIN
  SET @d = 0;
  SELECT disponibilità INTO @d FROM biglietto
     WHERE (biglietto.id = NEW.biglietto);
  IF (@d > 0) THEN
     UPDATE biglietto
     SET disponibilità = disponibilità - 1 WHERE id = NEW.biglietto;
  ELSE signal sqlstate '45010' SET message_text = 'Non disponibile';
  END IF;
END; $$
DELIMITER;

CREATE VIEW listarivenditori
AS SELECT nome, CONCAT (via,',', numero, ',', città) as indirizzo, telefono, email
FROM rivenditore;

DELIMITER $$
CREATE PROCEDURE sp_get10BestRivenditori(IN mese VARCHAR(45),IN anno
VARCHAR(45))
BEGIN
  SELECT r1.nome, count(*) as numeroVendite
  FROM rivendita r2 INNER JOIN rivenditore r1 ON r2.rivenditore = r1.codice
  WHERE (month(datacessione) = mese AND year(datacessione) = anno)
  GROUP BY r2.rivenditore
  ORDER BY numeroVendite DESC
  LIMIT 10;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_getCustTodayBirthday()
BEGIN
  SELECT count(*) INTO @x FROM spettatore WHERE (month(datanascita) = month(now()) AND day(datanascita) = day(now()));
  IF (@x = 0) THEN signal sqlstate '45101' SET message_text = 'Nessun risultato trovato';
  END IF;
  SELECT nome, cognome, email, telefono FROM spettatore
  WHERE month(datanascita) = month(now()) AND day(datanascita) = day(now());
END;$$
DELIMITER;

DELIMITER $$
CREATE FUNCTION udf_bigl_status( bigl INT(11))
RETURNS VARCHAR(45) DETERMINISTIC
BEGIN
    DECLARE var INT(11);
    DECLARE stato VARCHAR(45);
    SELECT disponibilità INTO var FROM biglietto
        WHERE id = bigl;
    IF (var > 0) THEN SET stato = 'Disponibile';
        ELSE SET stato = 'Esaurito';
    END IF;
    RETURN stato;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_getBigliettiConcertiCantante(IN singer VARCHAR(45))
BEGIN
  SELECT count(*) INTO @x FROM concerto WHERE (artista = singer);
  IF (@x = 0) THEN signal sqlstate '45101' SET message_text = 'Nessun risultato per il cantante indicato';
  END IF;
  SELECT data, ora, arena, artista, costo as prezzo,
         (SELECT udf_bigl_status(b.id)) as stato
  FROM concerto c
  INNER JOIN biglietto b ON c.biglietto = b.id
  WHERE artista = singer
  ORDER BY data;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_getBigliettiConcertiGenere (IN gen VARCHAR(45))
BEGIN
  SELECT c2.data,c2.ora,c2.arena,c2.artista,c1.genere,b.costo as prezzo,
          (SELECT udf_bigl_status(b.id)) as stato
  FROM cantante c1 INNER JOIN concerto c2 on c2.artista = c1.nome
  INNER JOIN biglietto b ON c2.biglietto = b.id
  WHERE c1.genere = gen
  ORDER BY c2.data;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_getBigliettiPartitediSport (IN disciplina VARCHAR(45))
BEGIN
  SELECT data, ora, arena, competizione, CONCAT(locali,'-',ospiti) as partita, costo as prezzo,
          (SELECT udf_bigl_status(b.id)) as stato
  FROM partita p INNER JOIN biglietto b ON p.biglietto = b.id
  WHERE (SELECT sport FROM squadra WHERE nome = locali) = disciplina
  ORDER BY data;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_getBigliettiPartiteSquadra(IN squad VARCHAR(45))
BEGIN
  SELECT count(*) INTO @x FROM partita WHERE (locali = squad OR ospiti = squad);
  IF (@x = 0) THEN signal sqlstate '45101' SET message_text = 'Nessun risultato per la squadra indicata';
  END IF;
  SELECT data, ora, arena, competizione, sport, CONCAT(locali,'-',ospiti) as partita,
       costo as prezzo, (SELECT udf_bigl_status(b.id)) as stato FROM biglietto b
  INNER JOIN partita p ON b.id = p.biglietto
  INNER JOIN squadra s ON p.locali = s.nome
  WHERE locali = squad OR ospiti = squad
  ORDER BY data;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_EmailPhoneList (INOUT email_list varchar(4000), INOUT
phone_list varchar(4000))
BEGIN
  DECLARE finished INTEGER DEFAULT 0;
  DECLARE v_email varchar(100) DEFAULT '';
  DECLARE v_phone varchar(100) DEFAULT '';
  DECLARE email_cursor CURSOR FOR SELECT email FROM spettatore;
  DECLARE phone_cursor CURSOR FOR SELECT telefono FROM spettatore;
  DECLARE CONTINUE HANDLER FOR NOT FOUND SET finished = 1;
  OPEN email_cursor;
  OPEN phone_cursor;
  WHILE (finished = 0) DO
       FETCH email_cursor INTO v_email;
       FETCH phone_cursor INTO v_phone;
       IF finished = 0 THEN
           SET email_list = CONCAT(email_list, v_email, '; ');
           SET phone_list = CONCAT(phone_list, v_phone, '; ');
       END IF;
  END WHILE;
  CLOSE email_cursor;
  CLOSE phone_cursor;
END;$$
DELIMITER;

DELIMITER $$
CREATE PROCEDURE sp_SoldOut()
BEGIN
    SELECT b.id as numBiglietto, CONCAT('idPartita: ',p.id) as IDevento ,CONCAT(locali,'-',ospiti) as evento, disponibilità
    FROM biglietto b INNER JOIN partita p ON p.biglietto = b.id
    WHERE b.disponibilità = 0
    UNION
    SELECT b.id as numBiglietto,CONCAT('idConcerto: ',c.id) as IDevento, CONCAT('Concerto ',c.artista) as evento, disponibilità
    FROM biglietto b INNER JOIN concerto c ON b.id = c.biglietto
    WHERE b.disponibilità = 0;
END; $$
DELIMITER;

INSERT INTO arena
VALUES ('Stadio Diego Armando Maradona','via Insigne',2,'Napoli',54726,'Aperto'),
('Stadio San Nicola','via S.Nicola',11,'Bari',58270,'Aperto'),
('Stadio Artemio Franchi','via Vittoria',6,'Firenze',43147,'Aperto'),
('Allianz Stadium','via Del Piero',14,'Torino',41507,'Aperto'),
('Stadio Arechi','via Bonazzoli ',6,'Salerno',37180,'Aperto'),
('Stadio Luigi Ferraris','via Mimmo Criscito', 71,'Genova',36599,'Aperto'),
('Stadio Renato Dall''Ara','via del tortellino',12,'Bologna',36462,'Aperto'),
('Stadio Bentegodi','via veneta',16,'Verona',31045,'Aperto'),
('Stadio Olimpico Grande Torino','via toro',1,'Torino',28177,'Aperto'),
('Stadio Nereo Rocco','piazzale Atleti Azzurri',2,'Trieste',26500,'Aperto'),
('Stadio Atleti Azzurri','piazza Gomez',13,'Bergamo',22512,'Aperto'),
('Mapei Stadium','via stadio',1,'Reggio Emilia',21525,'Aperto'),
('Sardegna Arena','viale dei leoni',2,'Cagliari',16416,'Aperto'),
('Stadio Picco','via dei campi',11,'La Spezia',11512,'Aperto'),
('Stadio Penzo','viale S.Pietro',1,'Venezia',11150,'Aperto'),
('Stadio San Siro','piazzale Angelo Moratti',1,'Milano',82000,'Aperto'),
('Piazza Duomo','piazza Duomo',null,'Milano', 20000,'Aperto'),
('Piazza Plebiscito','piazza Plebiscito',null,'Napoli', 25000,'Aperto'),
('Piazza San Marco','piazza San Marco',null,'Venezia', 10000,'Aperto'),
('Piazza Unità d''Italia','piazza Unità d''Italia',null,'Trieste', 10000,'Aperto'),
('Piazza San Carlo','piazza San Carlo',null,'Torino', 10000,'Aperto'),
('Arena Verona','via Pellisier',1,'Verona', 13000,'Aperto'),
('Anfiteatro La Civitella','via Marangoni',1,'Chieti', 10000,'Aperto'),
('Villa Manin','piazza Manin',10,'Codroipo', 5000,'Aperto'),
('Rimini Beach Arena','viale Principe Piemonte',56,'Rimini', 10000,'Aperto'),
('Arena Alpe Adria','via Adriatica',1,'Lignano Sabbiadoro', 5000,'Aperto'),
('Riccione Beach Arena','via Torino',null,'Riccione', 6000,'Aperto'),
('Unipol Arena','via Pajola',11,'Bologna', 18000,'Chiuso'),
('Palazzo dello Sport','via Datome',8,'Roma', 9000,'Chiuso'),
('PalaDesio','via Cipolla',1,'Desio', 6000,'Chiuso'),
('PalaDozza','via Comuzzo',2,'Bologna', 8000,'Chiuso'),
('PalaLeonessa','via Tonali',1,'Brescia', 5000,'Chiuso'),
('Taliercio','via Tonut',1,'Mestre', 3000,'Chiuso'),
('PalaEstra','via Pianigiani',2,'Siena', 6000,'Chiuso'),
('Kioene Arena','via Andreaus','4','Padova',8000,'Chiuso'),
('Circo Massimo','via Totti','10','Roma',100000,'Aperto'),
('Stade de France','Saint-Denis','1','Parigi','80000','Aperto'),
('Mediolanum Forum','via Meneghin','4','Milano',12000,'Chiuso'),
('Palazzetto Camagna','piazzale Carnera',1,'Tortona',6000,'Chiuso'),
('Palaserradimigni','piazza Segni',1,'Sassari',5000,'Chiuso'),
('PalaSport Pesaro','via crociera',71,'Pesaro',6000,'Chiuso'),
('Stadio Castellani','via Caracciolo',2,'Empoli',20000,'Aperto'),
('Dacia Arena','via Guidolin',11,'Udine',25000,'Aperto'),
('Stadio Olimpico','via Totti',10,'Roma',70000,'Aperto'),
('Allianz Dome','via Flavia',7,'Trieste',7000,'Chiuso'),
('PalaMazzola','via atleti azzurri','16','Taranto',5000,'Chiuso'),
('Pala De Andrè','via Carducci','1','Ravenna',4500,'Chiuso'),
('BLM Group Arena','via Vespucci','10','Trento',4000,'Chiuso'),
('Pala Barton','via martiri della libertà','14','Perugia',3500,'Chiuso'),
('Arena Kombëtare','Albania street','21','Tirana',22000,'Aperto');

INSERT INTO squadra
VALUES ('Inter','Calcio','Milano'),('Milan','Calcio','Milano'),('Juventus','Calcio','Torino'),('Torino','Calcio','Torino'),
       ('Roma','Calcio','Roma'),('Lazio','Calcio','Roma'),('Atalanta','Calcio','Bergamo'),('Udinese','Calcio','Udine'),
       ('Fiorentina','Calcio','Firenze'),('Genoa','Calcio','Genova'),('Sampdoria','Calcio','Genova'),
       ('Salernitana','Calcio','Salerno'),('Bologna','Calcio','Bologna'),('Cagliari','Calcio','Cagliari'),
       ('Empoli','Calcio','Empoli'),('Sassuolo','Calcio','Reggio Emilia'),('Napoli','Calcio','Napoli'),
       ('Hellas Verona','Calcio','Hellas Verona'),('Venezia','Calcio','Venezia'),('Spezia','Calcio','La Spezia'),
       ('Sir Safety Conad Perugia','Pallavolo','Perugia'),('Cucine Lube Civitanova','Pallavolo','Civitanova'),
       ('Itas Trentino','Pallavolo','Trento'),('Leo Shoes Modena','Pallavolo','Modena'),
       ('Allianz Milano','Pallavolo','Milano'), ('Bluenergy Piacenza','Pallavolo','Piacenza'),
       ('Vero Volley Monza','Pallavolo','Monza'),('Top Volley Cisterna','Pallavolo','Cisterna di Latina'),
       ('Verona Volley','Pallavolo','Verona'),('Prisma Taranto','Pallavolo','Taranto'),
       ('Kioene Padova','Pallavolo','Padova'),('Callipo Calabria Vibo Valentia','Pallavolo','Vibo Valentia'),
       ('Consar Ravenna','Pallavolo','Ravenna'),('Olimpia Milano','Basket','Milano'),
       ('Virtus Bologna','Basket','Bologna'),('Fortitudo','Basket','Bologna'),('Allianz Trieste','Basket','Trieste'),
       ('Dinamo Sassari','Basket','Sassari'),('Umana Reyer','Basket','Venezia'),('Dolomiti Trento','Basket','Trento'),
       ('Leonessa Brescia','Basket','Brescia'),('Derthona','Basket','Tortona'),('Reggiana','Basket','Reggio Emilia'),
       ('Universo Treviso','Basket','Treviso'), ('VL Pesaro','Basket','Pesaro'),('HappyCasa Brindisi','Basket','Brindisi'),
       ('Pallacanestro Varese','Basket','Varese'),('Napoli Basket','Basket','Napoli'), ('Vanoli Cremona','Basket','Cremona'),
       ('Real Madrid','Calcio','Madrid'),('FC Barcelona','Calcio','Barcellona'),('Liverpool','Calcio','Liverpool'),
       ('Chelsea','Calcio','Londra'),('Arsenal','Calcio','Londra'),('Tottenham','Calcio','Londra'),
       ('Manchester United','Calcio','Manchester'),('Manchester City','Calcio','Manchester'),('Paris Saint Germain','Calcio','Parigi'),
       ('Villareal','Calcio','Villareal'),('Atletico Madrid','Calcio','Madrid'),('Bayern Monaco','Calcio','Monaco di Baviera'),
       ('Borussia Dortmund','Calcio','Dortmund'),('Zenit','Calcio','San Pietroburgo'),('Ajax','Calcio','Amsterdam'),
       ('Galatasaray','Calcio','Istanbul'),('Sevilla','Calcio','Siviglia'),('Benfica','Calcio','Lisbona'),
       ('Leipzig','Calcio','Lipsia'),('Wolfsburg','Calcio','Wolfsburg'),('Porto','Calcio','Oporto'),
       ('Sporting','Calcio','Lisbona'),('Salzburg','Calcio','Salisburgo'),('Club Brugge','Calcio','Bruges'),
       ('Dinamo Kyiv','Calcio','Kiev'),('Shaktar Donetsk','Calcio','Donetsk'),('Besiktas','Calcio','Istanbul'),
       ('Young Boys','Calcio','Berna'),('Malmo','Calcio','Malmo'),('Sheriff','Calcio','Tiraspol'),
       ('Feyenoord','Calcio','Rotterdam');

INSERT INTO cantante
VALUES ('Eminem','Rap','USA'),('Fabri Fibra','Rap','Italia'),('Sfera Ebbasta','Rap','Italia'),
       ('Madame','Rap','Italia'), ('Ernia','Rap','Italia'), ('Salmo','Rap','Italia'),('Ghali','Rap','Italia'),
       ('Blanco','Rap','Italia'),('Elisa','Pop','Italia'),('Tommaso Paradiso','Pop','Italia'),('Jovanotti','Pop','Italia'),
       ('Laura Pausini','Pop','Italia'),('Emma','Pop','Italia'),('Baby K','Rap','Italia'),('Fedez','Rap','Italia'),
       ('J-AX','Rap','Italia'),('Maneskin','Rock','Italia'),('Vasco Rossi','Rock','Italia'),('AC-DC','Rock','Australia'),
       ('Bruce Springsteen','Rock','USA'),('Metallica','Rock','USA'),('Imagine Dragons','Rock','USA'),
       ('Rolling Stones','Rock','Regno Unito'),('James Blunt','Pop','Regno Unito'),('Ed Sheeran','Pop','Regno Unito'),
       ('Green Day','Pop','USA'),('Justin Timberlake','Pop','USA'),('Justin Bieber','Pop','Canada'),
       ('Achille Lauro','Pop','Italia'),('Adele','Pop','USA'),('Alvaro Soler','Pop','Spagna'),
       ('Avril Lavigne','Pop','USA'),('Bruno Mars','Pop','USA'),('Biagio Antonacci','Pop','Italia'),
       ('Coldplay','Pop','USA'),('Lady Gaga','Pop','USA'), ('Beyonce','Pop','USA'),('Rihanna','Pop','USA'),
       ('Macklemore','Rap','USA'),('Snoop Dog','Rap','USA'),('U2','Rock','Irlanda'),('50 Cent','Rap','USA'),
       ('Rkomi','Rap','Italia'),('Billie Eilish','Pop','USA');

INSERT INTO biglietto
VALUES (null,20.00 , 15000),(null,21.00 , 14000),(null,15.00 , 17652), (null,25.00, 42983),(null,15.00 , 17000 ),
       (null,30.00 , 40001),(null,40.00, 57891 ),(null,45.00, 12349),(null,22.00, 21000),(null,35.00 ,27654 ),
       (null, 24.00,19876),(null,15.00, 18790),(null, 20.00, 18907),(null, 40.00, 25438 ),(null, 25.00 , 41092 ),
       (null,30.00 ,10982 ),(null,45.00,52037 ),(null,50.00 , 9807 ),(null,10.00 , 10200 ),(null,10.00 , 14780),
       (null,100.00 , 60000),(null,130.00 , 62000 ),(null,300.00 , 75000 ),(null,18.00, 9000 ),(null,18.00, 9000),
       (null,20.00, 4000 ),(null,15.00 , 3800),(null,15.00 ,3800 ),(null,16.00 , 4000),(null,25.00 , 2300 ),
       (null,25.00 , 2300),(null,22.00 , 3000),(null,30.00 , 6000 ),(null,30.00 , 6000), (null,28.00 , 5000),
       (null,80.00, 9000 ), (null,78.00, 11000),(null,60.00, 14000 ),(null,25.00 , 3800),(null,80.00 ,3800 ),
       (null,86.00 , 24000),(null,90.00 , 22300 ),(null,60.00 , 2300),(null,25.00 , 3000),(null,70.00 , 26000 ),
       (null,25.00 , 2300),(null,25.00, 2000 ),(null,22.00, 9000),(null,30.00, 24000 ),(null,30.00 , 5800),
       (null,25.00 ,3800 ),(null,25.00 , 3000),(null,25.00 , 2300 ),(null,24.00 , 2300),(null,22.00 , 2000),
       (null,25.00 , 9000 ),(null,25.00 , 10000),(null,30.00, 29000 ),(null,60.00, 9000),(null,25.00, 3000 ),
       (null,25.00 , 3800),(null,25.00 ,3800 ), (null,30.00 , 24000),(null,25.00 , 2300 ),(null,25.00 , 2300),
       (null,24.00 , 3000),(null,25.00 , 3000 ),(null,82.00 , 46000),(null,28.00, 9000 ),(null,28.00, 3000),
       (null,26.00, 4000 ),(null,27.00 , 13800),(null,25.00 , 8800 ),(null,86.00 , 44000),(null,60.00 , 2300 ),
       (null,75.00 , 22300),(null,78.00 , 23000),(null,80.00 , 26000 ),(null,70.00 , 6000),(null,68.00, 9000 ),
       (null,58.00, 4000),(null,50.00, 4000 ),(null,55.00 , 3800),(null,25.00 ,3800 ),(null,26.00 , 14000),
       (null,40.00 , 22300 ),(null,81.00,40000),(null,74.00,12000),(null,66.00,8700),(null,102.00, 51000),
       (null,40.00,0),(null,15.50,1000),(null,16.00,1200),(null,14.00,1300),(null,18.00,2100),
       (null,17.50,1300),(null,13.00,1500),(null, 30.00, 0),(null, 40.00, 0),(null, 35.00, 0);

INSERT INTO partita
VALUES (null,'2022-06-14','15:00','Stadio Castellani','Serie A','Empoli','Salernitana',1),
       (null,'2022-06-14','18:00','Stadio Bentegodi','Serie A','Hellas Verona','Torino',2),
       (null,'2022-06-14','18:00','Dacia Arena','Serie A','Udinese','Spezia',3),
       (null,'2022-06-14','21:00','Stadio Olimpico','Serie A','Roma','Venezia',4),
       (null,'2022-06-15','12:30','Stadio Renato Dall''Ara','Serie A','Bologna','Sassuolo',5),
       (null,'2022-06-15','15:00','Stadio Diego Armando Maradona','Serie A','Napoli','Genoa',6),
       (null,'2022-06-15','18:00','Stadio San Siro','Serie A','Milan','Atalanta',7),
       (null,'2022-06-16','21:00','Sardegna Arena','Serie A','Cagliari','Inter',8),
       (null,'2022-06-16','19:00','Stadio Luigi Ferraris','Serie A','Sampdoria','Fiorentina',9),
       (null,'2022-06-15','21:00','Allianz Stadium','Serie A','Juventus','Lazio',10),
       (null,'2022-06-20','21:00','Stadio Olimpico Grande Torino','Serie A','Torino','Roma',11),
       (null,'2022-06-21','17:00','Stadio Luigi Ferraris','Serie A','Genoa','Bologna',12),
       (null,'2022-06-21','21:00','Stadio Atleti Azzurri','Serie A','Atalanta','Empoli',13),
       (null,'2022-06-21','21:00','Stadio Artemio Franchi','Serie A','Fiorentina','Juventus',14),
       (null,'2022-06-21','21:00','Stadio Olimpico','Serie A','Lazio','Hellas Verona',15),
       (null,'2022-06-22','12:30','Stadio Picco','Serie A','Spezia','Napoli',16),
       (null,'2022-06-22','18:00','Stadio San Siro','Serie A','Inter','Sampdoria',17),
       (null,'2022-06-22','18:00','Mapei Stadium','Serie A','Sassuolo','Milan',18),
       (null,'2022-06-22','21:00','Stadio Penzo','Serie A','Venezia','Cagliari',19),
       (null,'2022-06-22','21:00','Stadio Arechi','Serie A','Salernitana','Udinese',20),
       (null,'2022-08-10','21:00','Stadio Olimpico','Supercoppa','Milan','Inter',21),
       (null,'2022-06-11','21:00','Stadio Olimpico','Coppa Italia','Juventus','Inter',22),
       (null,'2022-06-28','21:00','Stade de France','Champions League','Liverpool','Real Madrid',23),
       (null,'2022-06-14','18:00','Unipol Arena','LegaBasketA','Virtus Bologna','VL Pesaro',24),
       (null,'2022-06-16','21:00','Unipol Arena','LegaBasketA','Virtus Bologna','VL Pesaro',25),
       (null,'2022-06-18','18:00','PalaSport Pesaro','LegaBasketA','VL Pesaro','Virtus Bologna',26),
       (null,'2022-06-14','20:00','PalaLeonessa','LegaBasketA','Leonessa Brescia','Dinamo Sassari',27),
       (null,'2022-06-16','20:00','PalaLeonessa','LegaBasketA','Leonessa Brescia','Dinamo Sassari',28),
       (null,'2022-06-18','18:00','Palaserradimigni','LegaBasketA','Dinamo Sassari','Leonessa Brescia',29),
       (null,'2022-06-15','18:00','Taliercio','LegaBasketA','Umana Reyer','Derthona',30),
       (null,'2022-06-17','20:00','Taliercio','LegaBasketA','Umana Reyer','Derthona',31),
       (null,'2022-06-19','18:00','Palazzetto Camagna','LegaBasketA','Derthona','Umana Reyer',32),
       (null,'2022-06-15','20:00','Mediolanum Forum','LegaBasketA','Olimpia Milano','Allianz Trieste',33),
       (null,'2022-06-17','18:00','Mediolanum Forum','LegaBasketA','Olimpia Milano','Allianz Trieste',34),
       (null,'2022-06-19','20:00','Allianz Dome','LegaBasketA','Allianz Trieste','Olimpia Milano',35),
       (null,'2022-06-02','19:30','Stade de France','Amichevole','Paris Saint Germain','Chelsea',91),
       (null,'2022-06-21','19:30','Kioene Arena','SuperLega','Kioene Padova','Callipo Calabria Vibo Valentia',92),
       (null,'2022-06-21','20:30','Mediolanum Forum','SuperLega','Allianz Milano','Cucine Lube Civitanova',93),
       (null,'2022-06-21','20:00','PalaMazzola','SuperLega','Prisma Taranto','Bluenergy Piacenza',94),
       (null,'2022-06-22','21:00','BLM Group Arena','SuperLega','Itas Trentino','Leo Shoes Modena',95),
       (null,'2022-06-22','20:30','Pala De Andrè','SuperLega','Consar Ravenna','Verona Volley',96),
       (null,'2022-06-22','18:30','Pala Barton','SuperLega','Sir Safety Conad Perugia','Vero Volley Monza',97),
       (null,'2022-06-08','20:45','Arena Kombëtare','Conference League','Roma','Feyenoord',100);

INSERT INTO concerto
VALUES (null,'2022-06-14','20:00','Mediolanum Forum','Eminem',36),
       (null,'2022-06-30','21:00','Stadio Olimpico Grande Torino','Imagine Dragons',37),
       (null,'2022-07-18','20:30','Stadio Renato Dall''Ara','Metallica',38),
       (null,'2022-08-20','21:00','Arena Verona','Laura Pausini',39),
       (null,'2022-06-16','21:00','Unipol Arena','Eminem',40),
       (null,'2022-09-14','20:30','Stadio San Siro','Green Day',41),
       (null,'2022-08-02','21:00','Stadio Olimpico','Lady Gaga',42),
       (null,'2022-06-22','21:30','Mediolanum Forum','James Blunt',43),
       (null,'2022-06-30','20:00','Kioene Arena','Emma',44),
       (null,'2022-09-01','20:30','Stadio San Siro','Bruce Springsteen',45),
       (null,'2022-06-30','21:00','Arena Alpe Adria','Elisa',46),
       (null,'2022-07-15','20:00','Piazza Unità d''Italia','Elisa',47),
       (null,'2022-07-22','20:45','Mapei Stadium','Elisa',48),
       (null,'2022-06-24','21:00','Stadio San Siro','Tommaso Paradiso',49),
       (null,'2022-06-27','20:30','Arena Verona','Tommaso Paradiso',50),
       (null,'2022-07-07','21:30','Arena Alpe Adria','Tommaso Paradiso',51),
       (null,'2022-07-30','21:00','Riccione Beach Arena','Tommaso Paradiso',52),
       (null,'2022-07-21','20:30','Rimini Beach Arena','Tommaso Paradiso',53),
       (null,'2022-07-13','21:30','Villa Manin','Madame',54),
       (null,'2022-06-30','21:30','Anfiteatro La Civitella','Madame',55),
       (null,'2022-07-27','21:00','Stadio Renato Dall''Ara','Madame',56),
       (null,'2022-07-20','21:00','Stadio Bentegodi','Madame',57),
       (null,'2022-08-01','21:00','Stadio San Siro','Fabri Fibra',58),
       (null,'2022-07-27','20:30','Arena Verona','James Blunt',59),
       (null,'2022-07-11','21:30','Arena Alpe Adria','Fabri Fibra',60),
       (null,'2022-07-21','21:00','Riccione Beach Arena','Fabri Fibra',61),
       (null,'2022-07-27','20:30','Rimini Beach Arena','Fabri Fibra',62),
       (null,'2022-08-08','21:00','Stadio San Siro','Ernia',63),
       (null,'2022-08-05','20:30','Arena Verona','Ernia',64),
       (null,'2022-08-03','21:30','Arena Alpe Adria','Ernia',65),
       (null,'2022-08-10','21:00','Riccione Beach Arena','Ernia',66),
       (null,'2022-08-12','20:30','Rimini Beach Arena','Ernia',67),
       (null,'2022-08-30','21:00','Stadio San Siro','Justin Timberlake',68),
       (null,'2022-07-19','21:30','Arena Verona','Blanco',69),
       (null,'2022-07-07','21:30','Arena Alpe Adria','Blanco',70),
       (null,'2022-07-30','21:00','Sardegna Arena','Blanco',71),
       (null,'2022-07-21','21:30','Stadio Artemio Franchi','Blanco',72),
       (null,'2022-08-01','21:30','Stadio Castellani','Blanco',73),
       (null,'2022-07-14','21:30','Stadio San Siro','Lady Gaga',74),
       (null,'2022-07-28','21:00','Kioene Arena','James Blunt',75),
       (null,'2022-06-18','21:30','Stadio San Siro','Billie Eilish',76),
       (null,'2022-07-15','21:30','Circo Massimo','Maneskin',77),
       (null,'2022-07-18','21:00','Stadio San Siro','Maneskin',78),
       (null,'2022-09-30','21:00','Unipol Arena','Maneskin',79),
       (null,'2022-09-01','21:30','Arena Verona','Maneskin',80),
       (null,'2022-08-30','20:00','Arena Alpe Adria','Jovanotti',81),
       (null,'2022-08-28','20:00','Riccione Beach Arena','Jovanotti',82),
       (null,'2022-08-26','20:30','Rimini Beach Arena','Jovanotti',83),
       (null,'2022-10-07','20:30','Kioene Arena','Ghali',84),
       (null,'2022-07-25','21:00','Piazza Duomo','Rkomi',85),
       (null,'2022-07-30','21:30','Circo Massimo','Sfera Ebbasta',86),
       (null,'2022-08-03','20:00','Stadio Olimpico','AC-DC',87),
       (null,'2022-08-26','20:30','Stadio Renato Dall''Ara','U2',88),
       (null,'2022-10-01','21:30','Arena Verona','Vasco Rossi',89),
       (null,'2022-09-10','20:00','Stadio San Siro','Rolling Stones',90),
       (null,'2022-06-03','21:00','Arena Verona','Coldplay',98),
       (null,'2022-06-05','21:30','Unipol Arena','U2',99);

INSERT INTO spettatore
VALUES  ('Stefano','Poni','STFPON64E05F205X','1964-06-11','Brindisi','3390877205','stefanoponi@gmail.it'),
        ('Ciro','Esposito','CROESP96E05F205X','1996-02-24','Napoli','3334872095','ciroespo@gmail.it'),
        ('Ludovica','Pagani','LDVPGN92E05F205X','1992-09-21','Brescia','3384772096','ludopaga@gmail.it'),
        ('Marco','Matrics','MRCMTR73E05F205X','1973-07-10','Perugia','3384772091','matrix23@gmail.it'),
        ('Elisabetta','Canna','ELSCAN80E05F205X','1980-10-20','Milano','3384772099','elisabethcann@gmail.it'),
        ('Mara','Donna','MARDNA58E05F205X','1958-09-04','Avellino','3384772081','maradonnadedios@gmail.com'),
        ('Enrico','Pappino','ENCPPN70E05F205X','1970-05-14','Pisa','3344772001','enricoreal@gmail.it'),
        ('Laura','Leopardi','LAULEO01E05F205X','2001-06-10','Torino','3334772042','lauretta01@gmail.it'),
        ('Simona','Di Biagio','SMNDBG86E05F205X','1986-11-18','Ancona','3395272095','simonadibi@libero.com'),
        ('Patrizia','Andreotti','PTRADR78E05F205X','1978-11-08','Roma','3334772095','patttty@libero.it'),
        ('Mimmo','Berardi','BRDMMM94E05F205X','1994-08-01','Reggio Emilia','3394472021','mimmobomber94@libero.it'),
        ('Francesco','Cozza','FRCCZZ74E05F205X','1974-09-08','Reggio Calabria','3384772095','cicciocozza@gmail.it'),
        ('Nicolò','Barella','NCLBRL97E05F205X','1997-04-28','Cagliari','3344767890','chebelgiocatore@gmail.it'),
        ('Andrea','Ranocchia','ANDRAN88E05F205X','1988-09-14','Bari','3384772097','andreafrog@gmail.it'),
        ('Roberto','Mancio','RBTMNC66E05F205X','1966-03-08','Jesi','3394702095','ilmancio@gmail.it'),
        ('Diletta','Leotta','DLTLTT91E05F205Y','1991-09-01','Palermo','3364942031','dilettanazionale@gmail.it'),
        ('Sara','Acquafresca','SARACQ89E05F205Y','1989-01-09','Genova','3384992095','saraacquafresh@gmail.com'),
        ('Martina','Giordani','MRTGRD94E05F205X','1994-11-05','Caserta','3474772095','lamarty@hotmail.it'),
        ('Filippo','Crosta','FLPCRS83E05F205X','1983-12-06','Parma','3384342095','pippocrosta@gmail.it'),
        ('Simone','Spiaze','SMNSPZ78E05F205X','1978-09-09','La Spezia','3384980707','simonespiaze@gmail.com'),
        ('Marta','Mei','MRTMEI89E05F205X','1989-07-08','Taranto','3334772095','martamei@gmail.it'),
        ('Luca','Mastrangelo','LUCMAS78E05F205X','1978-08-23','Catanzaro','334567205','lucamastra@gmail.it'),
        ('Piero','Augello','PIRAUG96E05F205X','1996-02-27','Torino','3334872095','piero.augello@gmail.it'),
        ('Maria','Peppina','MARPEP62E05F205X','1962-09-02','Catania','3398072531','mariapeppina@libero.it'),
        ('Lara','Delvecchio','LARDVC98E05F205Y','1998-01-10','Ancona','3334533097','ladelvecchio@outlook.com'),
        ('Giorgia','Trincheri','GRGTRC94E05F205Y','1994-05-31','Udine','3334562099','giorgetta@gmail.com'),
        ('Giulia','Franconi','GIUFRC88E05F205Y','1988-07-04','Cesena','3345678909','gfranconi@virgilio.it'),
        ('Carolina','Amici','CRLAMC99E05F205Y','1999-08-18','Lecce','3388802001','carolamici@outlook.it'),
        ('Cinzia','Buscatti','CNZBST67E05F205X','1967-06-10','Padova','3424772042','cinziab@libero.it'),
        ('Gaia','Natali','GAINTL96E05F205X','1996-10-18','Ferrara','3389072095','gnatali@libero.it'),
        ('Marco','Natali','MRCNTL94E05F205X','1994-03-11','Ferrara','3383244507','mnatali@libero.it'),
        ('Michele','Cesari','MICCSR81E05F205X','1981-04-08','Perugia','3489072095','michelecesari@libero.it'),
       ('Elison','Di Benedetto','ELSDBN92E05F205X','1992-10-10','Teramo','3345602095','laelison@libero.it'),
       ('Mattia','Corazza','MTTCRZ77E05F205X','1977-05-22','Pistoia','3279852021','mattiacorazza@gmail.it'),
       ('Simone','Sandron','SMNSND97E05F205X','1997-05-22','Bergamo','3939852087','ssandron@libero.it'),
       ('Martina','Stella','MRTSTL92E05F205X','1992-05-23','Roma','3399852021','mstella@gmail.it'),
       ('Laura','Santangelo','LAUSTG93E05F205X','1993-05-23','Siena','3475467808','laurasangelo@libero.it'),
       ('Sara','Ercolano','SARERC8705F205X','1987-05-24','Siracusa','3275467098','saraercolano@gmail.it'),
       ('Matteo','Solidoro','MTTSLD97E05F205X','1997-05-25','Vicenza','3939823456','msolidoro@libero.it'),
       ('Filippo','Liverani','FLPLVR88E05F205X','1988-05-26','Palermo','329876543','pippoliverani@gmail.it'),
       ('Leonardo','Da Re','LNDDAR90E05F205X','1990-05-26','Firenze','3234567899','leodare@libero.it'),
       ('Lorenzo','Giannotti','LRZGNT84E05F205X','1984-05-27','Monza','3234567899','lollogiannetti@gmail.it'),
       ('Rebecca','Trottola','RBCTRT91E05F205X','1991-05-27','Genova','3234567707','beccatrottola@libero.it'),
       ('Patrizia','Ventola','PRZVTL68E05F205X','1968-05-28','Cuneo','3279852555','pattyventola@gmail.it'),
       ('Nicole','Santoro','NCLSNT87E05F205X','1987-05-28','Empoli','3939852666','nikisantoro@libero.it'),
       ('Sofia','Politano','SOFPLT01E05F205X','2001-05-29','Venezia','3279852999','sofiapolitano@gmail.it'),
       ('Alice','Tondo','ALCTND02E05F205X','2002-05-29','Milano','3939852121','alicetondo@libero.it'),
       ('Annalisa','D''Ercole','ANLDRC99E05F205X','1999-05-30','Frosinone','3279852444','annalisa.dercole@gmail.it'),
       ('Vanessa','Struzzo','VNSSRZ92E05F205X','1992-05-30','Imperia','3939852333','vane.struzzo@libero.it'),
       ('Valentina','Palladino','VLTPLD94E05F205X','1994-05-31','Reggio Emilia','3279852202','valepalladino@gmail.it'),
       ('Stefano','Chiesa','STFCHI97E05F205X','1997-05-31','Bologna','3429852902','s.chiesa@libero.it'),
       ('Alessandro','Chesini','ALSCHE70E05F205X','1970-06-01','Trento','3279855137','alessandro_chesini@gmail.it'),
       ('Alex','Cocolet','ALXCCL95E05F205X','1995-06-01','Perugia','3345566778','cocoletalex@libero.it'),
       ('Virginia','Agnello','VRGAGL91E05F205X','1991-06-02','Palermo','3279988776','agnellovirgi@gmail.it'),
       ('Anna','Pero','ANNPER02E05F205X','2002-06-02','Pisa','3931122334','anna.pero@libero.it'),
       ('Enrico','Rigoni','ENRRGN96E05F205X','1996-06-03','Brescia','3275511448','enrico.rigoni@gmail.it'),
       ('Stefania','Girotondo','STFGRT00E05F205X','2000-06-03','Roma','3423028704','stefigirotondo@libero.it');

INSERT INTO rivenditore
VALUES (null,'Bar Emilio','via Petrarca','48','Modena','0440 712348','baremilio@info.it'),
       (null,'Bar 4 Venti','viale San Marco','6','Siena','046 754248','quattroventi@info.it'),
       (null,'Edicola Bisiaca','via Pertini','22','Monfalcone','0481 712348','edicolabisiaca@info.it'),
       (null,'TuttoBiglietto','via Dante Alighieri','40','Bologna','034 712348','tuttobiglietto@gmail.it'),
       (null,'Tickets4Fan','via Venezia','13','Cuneo','044 712348','tickets4fan@libero.it'),
       (null,'Giornali ma non solo','via Augusto','4','Genova','338 9747803','notonlyjournals@info.it'),
       (null,'Edicolandia','via Pocar','71','Udine','334 5202137','edicolandiaud@info.it'),
       (null,'Alla stazione','via della stazione','45','Cervignano del Friuli','0432 712348','allastazione@gmail.com'),
       (null,'Tuttoedipiù','via San Nicola','6','Bari','339 712348','tuttoedipiù@gmail.com'),
       (null,'Tabaccheria Centrale','piazza Borsellino','15','Perugia','040 712348','tabacchicentrale@libero.it'),
       (null,'Da Arturo','via Nizza','4','Catania','333 712348','arturos@gmail.it'),
       (null,'InterClub Cividale','via del ponte','11','Cividale del Friuli','0432 959201','interclubciv@inter.it'),
       (null,'Max Tabacchi','via delle rondini','4','Veronsa','334 712348','maxtabacchi@libero.it'),
       (null,'Tabacchino del Re','via XXV aprile','3','Parma','047 712348','tabacchino@virgilio.it'),
       (null,'Cartoljet','via I maggio','60','Monza','02 712348','cartoljet@info.it'),
       (null,'Cartolibreria La Rosa','viale XX settembre','99','Roma','034 712348','cartorosa@gmail.it'),
       (null,'Tarantella','viale Speranza','111','Firenze','338 712348','tarantella@tabacchi.it'),
       (null,'Cerignola Giornali','via Boccaccio','21','Cerignola','331 712348','cerignolaedicola@libero.it'),
       (null,'7su7','viale dei giardini ','109','Napoli','038 712348','settesusette@gmail.com'),
       (null,'QuiLoTrovi','via Vittorio Veneto','8','Catanzaro','333 712348','quilotrovi@gmail.com'),
       (null,'Ticketsardi','via Baiamonti','4','Cagliari','334 5712348','sardisiamonoi@tickets.it'),
       (null,'Everyday','viale marittimo','44','Riccione','336 712348','everyday@gmail.com'),
       (null,'BiglieTotti','via Montale','7','Roma','338 412348','biglietotti@libero.it'),
       (null,'Bar Giangi','via Roma','71','Milano','02 94590577','bargiangi@libero.com');


INSERT INTO rivendita
    VALUES(null,'20','2', now()),(null,'21','2', now()),(null,'22','2', now()),(null,'23','2', now()),(null,'24','2', now()),
           (null,'25','2', now()),(null,'26','2', now()),(null,'27','2', now()),(null,'28','2', now()),(null,'29','2', now()),
           (null,'29','3', now()),(null,'29','4', now()),(null,'29','4', now()),(null,'29','5', now()),(null,'29','6', now()),
           (null,'29','7', now()),(null,'30','8', now()),(null,'30','9', now()),(null,'30','10', now()),(null,'30','11', now()),
           (null,'30','12', now()),(null,'30','13', now()),(null,'44','14', now()),(null,'45','15', now()),(null,'45','10', now()),
           (null,'46','10', now()),(null,'47','10', now()),(null,'47','16', now()),(null,'48','10', now()),(null,'47','10', now()),
           (null,'47','16', now()),(null,'47','17', now()),(null,'47','18', now()),(null,'47','10', now()), (null,'47','18', now()),
           (null,'20','2', now()),(null,'21','2', now()),(null,'22','2', now()),(null,'23','2', now()),(null,'24','2', now()),
           (null,'25','2', now()),(null,'26','2', now()),(null,'27','2', now()),(null,'28','2', now()),(null,'29','2', now()),
           (null,'29','3', now()),(null,'29','4', now()),(null,'29','4', now()),(null,'29','5', now()),(null,'29','6', now()),
           (null,'29','7', now()),(null,'30','8', now()),(null,'30','9', now()),(null,'30','10', now()),(null,'30','11', now()),
           (null,'30','12', now()),(null,'30','13', now()),(null,'44','14', now()),(null,'45','15', now()),(null,'45','10', now()),
           (null,'46','10', now()),(null,'47','10', now()),(null,'47','16', now()),(null,'48','10', now()),(null,'47','10', now()),
           (null,'47','16', now()),(null,'47','17', now()),(null,'47','18', now()),(null,'47','10', now()), (null,'47','18', now()),
           (null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),
           (null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),(null,'21','2', now()),
           (null,'21','3', now()),(null,'21','4', now()),(null,'21','4', now()),(null,'21','5', now()),(null,'21','6', now()),
           (null,'22','7', now()),(null,'22','8', now()),(null,'30','9', now()),(null,'30','10', now()),(null,'30','11', now()),
           (null,'22','12', now()),(null,'22','13', now()),(null,'22','14', now()),(null,'22','15', now()),(null,'22','10', now()),
           (null,'22','10', now()),(null,'23','10', now()),(null,'23','16', now()),(null,'24','10', now()),(null,'22','10', now()),
           (null,'23','16', now()),(null,'23','17', now()),(null,'23','18', now()),(null,'24','10', now()), (null,'25','18', now()),
           (null,'90','23', now()),(null,'31','22', now()),(null,'32','22', now()),(null,'33','22', now()),(null,'44','21', now()),
           (null,'85','20', now()),(null,'46','20', now()),(null,'37','12', now()),(null,'38','12', now()),(null,'49','20', now()),
           (null,'79','13', now()),(null,'59','14', now()),(null,'39','14', now()),(null,'39','15', now()),(null,'49','16', now()),
           (null,'69','17', now()),(null,'80','18', now()),(null,'40','19', now()),(null,'40','19', now()),(null,'40','19', now()),
           (null,'50','22', now()),(null,'70','23', now()),(null,'34','14', now()),(null,'45','15', now()),(null,'45','19', now()),
           (null,'86','20', now()),(null,'67','20', now()),(null,'57','16', now()),(null,'48','20', now()),(null,'57','16', now()),
           (null,'77','16', now()),(null,'77','17', now()),(null,'37','18', now()),(null,'47','20', now()), (null,'71','18', now()),
           (null,'60','12', now()),(null,'71','12', now()),(null,'72','12', now()),(null,'83','12', now()),(null,'84','12', now()),
           (null,'65','13', now()),(null,'76','14', now()),(null,'77','13', now()),(null,'88','13', now()),(null,'89','13', now()),
           (null,'69','13', now()),(null,'79','14', now()),(null,'79','14', now()),(null,'89','15', now()),(null,'89','16', now()),
           (null,'69','17', now()),(null,'70','18', now()),(null,'70','19', now()),(null,'80','20', now()),(null,'80','21', now()),
           (null,'60','22', now()),(null,'70','23', now()),(null,'74','22', now()),(null,'85','20', now()),(null,'85','1', now()),
           (null,'66','10', now()),(null,'77','10', now()),(null,'77','16', now()),(null,'88','10', now()),(null,'87','10', now()),
           (null,'67','16', now()),(null,'76','17', now()),(null,'73','18', now()),(null,'87','10', now()), (null,'87','18', now()),
           (null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),
           (null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),(null,'8','24',now()),
           (null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),
           (null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),(null,'17','24',now()),
           (null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),
           (null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),(null,'21','24',now()),
           (null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),
           (null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),(null,'22','24',now()),
           (null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),
           (null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),(null,'43','24',now()),
           (null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),
           (null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),
           (null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),
           (null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),(null,'75','24',now()),
           (null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),
           (null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),(null,'23','24',now()),
           (null,'66','20', now()),(null,'75','16', now());

INSERT INTO acquisto
VALUES (null,'ANDRAN88E05F205X',1,now()),(null,'BRDMMM94E05F205X',2,now()),(null,'CNZBST67E05F205X',3,now()),
       (null,'CRLAMC99E05F205Y',4,now()),(null,'CROESP96E05F205X',5,now()),(null,'DLTLTT91E05F205Y',6,now()),
       (null,'ELSCAN80E05F205X',7,now()),(null,'ELSDBN92E05F205X',8,now()),(null,'ENCPPN70E05F205X',9,now()),
       (null,'FLPCRS83E05F205X',10,now()),(null,'FRCCZZ74E05F205X',11,now()),(null,'GAINTL96E05F205X',12,now()),
       (null,'GIUFRC88E05F205Y',13,now()),(null,'GRGTRC94E05F205Y',14,now()),(null,'LARDVC98E05F205Y',15,now()),
       (null,'LAULEO01E05F205X',16,now()),(null,'LDVPGN92E05F205X',17,now()),(null,'LUCMAS78E05F205X',18,now()),
       (null,'MARDNA58E05F205X',19,now()),(null,'MARPEP62E05F205X',20,now()),(null,'MICCSR81E05F205X',21,now()),
       (null,'MRCMTR73E05F205X',22,now()),(null,'MRCNTL94E05F205X',23,now()),(null,'MRTGRD94E05F205X',24,now()),
       (null,'MRTMEI89E05F205X',25,now()),(null,'NCLBRL97E05F205X',26,now()),(null,'PIRAUG96E05F205X',27,now()),
       (null,'PTRADR78E05F205X',28,now()),(null,'RBTMNC66E05F205X',29,now()),(null,'SARACQ89E05F205Y',30,now()),
       (null,'SMNDBG86E05F205X',31,now()),(null,'SMNSPZ78E05F205X',32,now()),(null,'STFPON64E05F205X',33,now()),
       (null,'SMNSPZ78E05F205X',43,now()),(null,'SMNSPZ78E05F205X',17,now()),(null,'SMNSPZ78E05F205X',22,now()),
       (null,'ANDRAN88E05F205X',75,now()),(null,'BRDMMM94E05F205X',43,now()),(null,'CNZBST67E05F205X',23,now()),
       (null,'CRLAMC99E05F205Y',75,now()),(null,'CROESP96E05F205X',43,now()),(null,'DLTLTT91E05F205Y',26,now()),
       (null,'ELSCAN80E05F205X',75,now()),(null,'ELSDBN92E05F205X',43,now()),(null,'ENCPPN70E05F205X',29,now()),
       (null,'FLPCRS83E05F205X',75,now()),(null,'FRCCZZ74E05F205X',43,now()),(null,'GAINTL96E05F205X',42,now()),
       (null,'GIUFRC88E05F205Y',75,now()),(null,'GRGTRC94E05F205Y',43,now()),(null,'LARDVC98E05F205Y',45,now()),
       (null,'LAULEO01E05F205X',75,now()),(null,'LDVPGN92E05F205X',43,now()),(null,'LUCMAS78E05F205X',48,now()),
       (null,'MARDNA58E05F205X',75,now()),(null,'MARPEP62E05F205X',43,now()),(null,'MICCSR81E05F205X',51,now()),
       (null,'MRCMTR73E05F205X',75,now()),(null,'MRCNTL94E05F205X',43,now()),(null,'MRTGRD94E05F205X',54,now()),
       (null,'MRTMEI89E05F205X',75,now()),(null,'NCLBRL97E05F205X',43,now()),(null,'PIRAUG96E05F205X',57,now()),
       (null,'PTRADR78E05F205X',75,now()),(null,'RBTMNC66E05F205X',43,now()),(null,'SARACQ89E05F205Y',60,now()),
       (null,'SMNDBG86E05F205X',75,now()),(null,'SMNSPZ78E05F205X',23,now()),(null,'STFPON64E05F205X',63,now());



