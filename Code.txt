-- logat ca sys

-- 1. Crearea utilizatorului OLTP

CREATE USER dwbi_oltp IDENTIFIED BY oltp;
GRANT CONNECT TO dwbi_oltp;
GRANT RESOURCE TO dwbi_oltp;
GRANT ALL ON DBMS_CRYPTO TO dwbi_oltp;
ALTER USER dwbi_oltp QUOTA UNLIMITED ON USERS;

-- 3. Crearea urilizatorului depozit

CREATE USER dwbi_olap IDENTIFIED BY olap;
GRANT CONNECT TO dwbi_olap;
GRANT RESOURCE TO dwbi_olap;
ALTER USER dwbi_olap QUOTA UNLIMITED ON USERS;
GRANT CREATE DIMENSION TO dwbi_olap;
GRANT CREATE MATERIALIZED VIEW TO dwbi_olap;
GRANT QUERY REWRITE TO dwbi_olap;

-----------------------------------------------------------------------------------------------------------------------------------------

-- logat ca dwbi_oltp

-- 1. Crearea bazei de date OLTP

CREATE TABLE regiune(
    id_regiune NUMBER PRIMARY KEY,
    nume VARCHAR2(50) NOT NULL
);

CREATE TABLE subregiune(
    id_subregiune NUMBER PRIMARY KEY,
    nume VARCHAR2(50) NOT NULL,
    id_regiune NUMBER NOT NULL REFERENCES regiune(id_regiune)
);

CREATE TABLE tara(
    id_tara NUMBER PRIMARY KEY,
    nume VARCHAR2(50) NOT NULL,
    id_subregiune NUMBER NOT NULL REFERENCES subregiune(id_subregiune)
);

CREATE TABLE post_tv( 
    id_post NUMBER PRIMARY KEY,
    denumire VARCHAR2(20) NOT NULL,
    id_tara NUMBER NOT NULL REFERENCES tara(id_tara),
    contract_transmisie_completa NUMBER(1) NOT NULL
);

CREATE TABLE jucator(
    id_jucator NUMBER PRIMARY KEY,
    nume VARCHAR2(25) NOT NULL,
    prenume VARCHAR2(25) NOT NULL,
    data_nastere DATE,
    id_tara NUMBER NOT NULL REFERENCES tara(id_tara),
    clasament NUMBER(4),
    inaltime NUMBER(3),
    greutate NUMBER(3),
    mana VARCHAR2(10),
    an_trecere_prof NUMBER(4),
    poza BLOB,
    cap_serie NUMBER(3),
    antrenor VARCHAR2(100)
);

CREATE TABLE sponsor(
    id_sponsor NUMBER PRIMARY KEY,
    denumire VARCHAR2(25) NOT NULL,
    logo BLOB NOT NULL, 
    link_site VARCHAR(1000) NOT NULL
);

CREATE TABLE contract(
    id_contract NUMBER PRIMARY KEY,
    id_sponsor NUMBER NOT NULL REFERENCES sponsor(id_sponsor) ON DELETE CASCADE,
    id_jucator NUMBER NOT NULL REFERENCES jucator(id_jucator) ON DELETE CASCADE,
    CONSTRAINT uq_contract UNIQUE (id_sponsor, id_jucator)
);

CREATE TABLE scor_meciuri_directe(
    id_smd NUMBER PRIMARY KEY,
    id_jucator1 NUMBER NOT NULL REFERENCES jucator(id_jucator) ON DELETE CASCADE,
    id_jucator2 NUMBER NOT NULL REFERENCES jucator(id_jucator) ON DELETE CASCADE,
    victorii_jucator1 NUMBER NOT NULL,
    victorii_jucator2 NUMBER NOT NULL,
    CONSTRAINT dif_juc_smd CHECK(id_jucator1 <> id_jucator2)
);

CREATE TABLE tablou( 
    id_tablou NUMBER PRIMARY KEY,
    tragere_sorti BLOB NOT NULL,
    data_tablou DATE DEFAULT SYSDATE
);

CREATE TABLE tur( 
    id_tur NUMBER PRIMARY KEY,
    denumire VARCHAR2(20) NOT NULL,
    pret NUMBER NOT NULL
);

CREATE TABLE teren( 
    id_teren NUMBER PRIMARY KEY,
    denumire VARCHAR2(20) NOT NULL,
    capacitate NUMBER(6)
);

CREATE TABLE arbitru( 
    id_arbitru NUMBER PRIMARY KEY,
    nume VARCHAR2(25) NOT NULL,
    prenume VARCHAR2(25) NOT NULL,
    id_tara NUMBER NOT NULL REFERENCES tara(id_tara)
);

CREATE TABLE meci(
    id_meci NUMBER PRIMARY KEY,
    id_jucator1 NUMBER REFERENCES jucator(id_jucator) ON DELETE SET NULL,
    id_jucator2 NUMBER REFERENCES jucator(id_jucator) ON DELETE SET NULL,
    id_teren NUMBER REFERENCES teren(id_teren) ON DELETE SET NULL,
    id_arbitru NUMBER REFERENCES arbitru(id_arbitru) ON DELETE SET NULL,
    id_tur NUMBER NOT NULL REFERENCES tur(id_tur) ON DELETE CASCADE,
    data_ora DATE,
    id_castigator NUMBER,
    scor VARCHAR2(20),
    durata NUMBER(3),
    locuri_disponibile NUMBER,
    CONSTRAINT dif_juc_meci CHECK(id_jucator1 <> id_jucator2),
    CONSTRAINT castigator_valid CHECK(id_castigator is null or id_castigator = id_jucator1 or id_castigator = id_jucator2)
);

CREATE TABLE user_(
    id_user NUMBER PRIMARY KEY,
    user_name VARCHAR2(255) NOT NULL,
    parola VARCHAR2(255) NOT NULL,
    email VARCHAR2(255) NOT NULL,
    data_nastere DATE NOT NULL,
    puncte NUMBER DEFAULT 0 NOT NULL,
    id_tara NUMBER NOT NULL REFERENCES tara(id_tara),
    CONSTRAINT uq_user UNIQUE (user_name),
    CONSTRAINT email_uq UNIQUE (email)
);

CREATE TABLE bilet(
    id_bilet NUMBER PRIMARY KEY,
    id_meci NOT NULL REFERENCES meci(id_meci),
    id_user NOT NULL REFERENCES user_(id_user),
    cantitate NUMBER NOT NULL,
    nr_comanda NUMBER NOT NULL,
    pret_platit NUMBER NOT NULL,
    data_achizitie DATE NOT NULL
);

CREATE TABLE subiect(
    id_subiect NUMBER PRIMARY KEY,
    titlu VARCHAR2(255) NOT NULL,
    descriere VARCHAR2(1000),
    id_user NOT NULL REFERENCES user_(id_user) ON DELETE CASCADE,
    data_postare date DEFAULT SYSDATE NOT NULL
);

CREATE TABLE comentariu(
    id_comentariu NUMBER PRIMARY KEY,
    continut VARCHAR2(4000) NOT NULL,
    id_subiect NOT NULL REFERENCES subiect(id_subiect) ON DELETE CASCADE,
    id_user NOT NULL REFERENCES user_(id_user) ON DELETE CASCADE,
    data_postare date DEFAULT SYSDATE NOT NULL,
    editat NUMBER(1) NOT NULL,
    CONSTRAINT bool_editat CHECK (editat = 0 OR editat = 1)
);

CREATE TABLE quiz(
    id_quiz NUMBER PRIMARY KEY,
    titlu VARCHAR2(255) NOT NULL,
    data_postare date DEFAULT SYSDATE UNIQUE NOT NULL
);

CREATE TABLE intrebare(
    id_intrebare NUMBER PRIMARY KEY,
    titlu VARCHAR2(3000) NOT NULL,
    raspuns1 VARCHAR2(2000) NOT NULL,
    raspuns2 VARCHAR2(2000) NOT NULL,
    raspuns3 VARCHAR2(2000) NOT NULL,
    raspuns4 VARCHAR2(2000) NOT NULL,
    raspuns_corect VARCHAR2(2000) NOT NULL,
    id_quiz NUMBER NOT NULL REFERENCES quiz(id_quiz) ON DELETE CASCADE,
    CONSTRAINT ex_rc CHECK (raspuns_corect = raspuns1 OR raspuns_corect = raspuns2 OR raspuns_corect = raspuns3 OR raspuns_corect = raspuns4)
);

CREATE TABLE scor_quiz(
    id_scor_quiz NUMBER PRIMARY KEY,
    id_user NUMBER NOT NULL REFERENCES user_(id_user) ON DELETE CASCADE,
    id_quiz NUMBER NOT NULL REFERENCES quiz(id_quiz) ON DELETE CASCADE,
    scor NUMBER,
    timp NUMBER,
    CONSTRAINT max_5_scor CHECK(scor <= 5),
    CONSTRAINT uq_user_quiz UNIQUE (id_user, id_quiz)
);

-- 2. Generarea datelor si inserarea acestora in tabele

INSERT INTO tur VALUES(1, '1st Round', 10);
INSERT INTO tur VALUES(2, '2nd Round', 25);
INSERT INTO tur VALUES(3, 'Quarter-Final', 75);
INSERT INTO tur VALUES(4, 'Semi-Final', 150);
INSERT INTO tur VALUES(5, 'Final', 400);

SELECT * FROM tur;

INSERT INTO teren VALUES(1, 'Court Rainer III', 10200);
INSERT INTO teren VALUES(2, 'Court des Princes', 1500);
INSERT INTO teren VALUES(3, 'Court No 2', 200);
INSERT INTO teren VALUES(4, 'Court No 9', 100);
INSERT INTO teren VALUES(5, 'Court No 11', 50);

SELECT * FROM teren;

INSERT INTO regiune VALUES (1, 'Europe');
INSERT INTO regiune VALUES (2, 'Asia');
INSERT INTO regiune VALUES (3, 'Africa');
INSERT INTO regiune VALUES (4, 'Americas');
INSERT INTO regiune VALUES (5, 'Oceania');
INSERT INTO regiune VALUES (6, 'Antarctica');

SELECT * FROM regiune;

INSERT INTO subregiune VALUES (1, 'Eastern Europe', 1);
INSERT INTO subregiune VALUES (2, 'Northern Europe', 1);
INSERT INTO subregiune VALUES (3, 'Southern Europe', 1);
INSERT INTO subregiune VALUES (4, 'Western Europe', 1);
INSERT INTO subregiune VALUES (5, 'Central Asia', 2);
INSERT INTO subregiune VALUES (6, 'Eastern Asia', 2);
INSERT INTO subregiune VALUES (7, 'South-eastern Asia', 2);
INSERT INTO subregiune VALUES (8, 'Southern Asia', 2);
INSERT INTO subregiune VALUES (9, 'Western Asia', 2);
INSERT INTO subregiune VALUES (10, 'Northern Africa', 3);
INSERT INTO subregiune VALUES (11, 'Eastern Africa', 3);
INSERT INTO subregiune VALUES (12, 'Middle Africa', 3);
INSERT INTO subregiune VALUES (13, 'Southern Africa', 3);
INSERT INTO subregiune VALUES (14, 'Western Africa', 3);
INSERT INTO subregiune VALUES (15, 'Caribbean', 4);
INSERT INTO subregiune VALUES (16, 'Central America', 4);
INSERT INTO subregiune VALUES (17, 'South America', 4);
INSERT INTO subregiune VALUES (18, 'Northern America', 4);
INSERT INTO subregiune VALUES (19, 'Australia and New Zealand', 5);
INSERT INTO subregiune VALUES (20, 'Melanesia', 5);
INSERT INTO subregiune VALUES (21, 'Micronesia', 5);
INSERT INTO subregiune VALUES (22, 'Polynesia', 5);

SELECT * FROM subregiune;

INSERT INTO tara VALUES (1, 'Afghanistan', 8);
INSERT INTO tara VALUES (2, 'Albania', 3);
INSERT INTO tara VALUES (3, 'Algeria', 10);
INSERT INTO tara VALUES (4, 'Andorra', 3);
INSERT INTO tara VALUES (5, 'Angola', 12);
INSERT INTO tara VALUES (6, 'Antigua and Barbuda', 15);
INSERT INTO tara VALUES (7, 'Argentina', 17);
INSERT INTO tara VALUES (8, 'Armenia', 9);
INSERT INTO tara VALUES (9, 'Australia', 19);
INSERT INTO tara VALUES (10, 'Austria', 4);
INSERT INTO tara VALUES (11, 'Azerbaijan', 9);
INSERT INTO tara VALUES (12, 'Bahamas', 15);
INSERT INTO tara VALUES (13, 'Bahrain', 9);
INSERT INTO tara VALUES (14, 'Bangladesh', 8);
INSERT INTO tara VALUES (15, 'Barbados', 15);
INSERT INTO tara VALUES (16, 'Belarus', 1);
INSERT INTO tara VALUES (17, 'Belgium', 4);
INSERT INTO tara VALUES (18, 'Belize', 16);
INSERT INTO tara VALUES (19, 'Benin', 14);
INSERT INTO tara VALUES (20, 'Bhutan', 8);
INSERT INTO tara VALUES (21, 'Bolivia', 17);
INSERT INTO tara VALUES (22, 'Bosnia and Herzegovina', 3);
INSERT INTO tara VALUES (23, 'Botswana', 13);
INSERT INTO tara VALUES (24, 'Brazil', 17);
INSERT INTO tara VALUES (25, 'Brunei', 7);
INSERT INTO tara VALUES (26, 'Bulgaria', 1);
INSERT INTO tara VALUES (27, 'Burkina Faso', 14);
INSERT INTO tara VALUES (28, 'Burundi', 11);
INSERT INTO tara VALUES (29, 'Cabo Verde', 14);
INSERT INTO tara VALUES (30, 'Cambodia', 7);
INSERT INTO tara VALUES (31, 'Cameroon', 12);
INSERT INTO tara VALUES (32, 'Canada', 18);
INSERT INTO tara VALUES (33, 'Central African Republic', 12);
INSERT INTO tara VALUES (34, 'Chad', 12);
INSERT INTO tara VALUES (35, 'Chile', 17);
INSERT INTO tara VALUES (36, 'China', 6);
INSERT INTO tara VALUES (37, 'Colombia', 17);
INSERT INTO tara VALUES (38, 'Comoros', 11);
INSERT INTO tara VALUES (39, 'Congo', 12);
INSERT INTO tara VALUES (40, 'Costa Rica', 16);
INSERT INTO tara VALUES (41, 'Croatia', 3);
INSERT INTO tara VALUES (42, 'Cuba', 15);
INSERT INTO tara VALUES (43, 'Cyprus', 9);
INSERT INTO tara VALUES (44, 'Czech Republic', 1);
INSERT INTO tara VALUES (45, 'Denmark', 2);
INSERT INTO tara VALUES (46, 'Djibouti', 11);
INSERT INTO tara VALUES (47, 'Dominica', 15);
INSERT INTO tara VALUES (48, 'Dominican Republic', 15);
INSERT INTO tara VALUES (49, 'East Timor', 7);
INSERT INTO tara VALUES (50, 'Ecuador', 17);
INSERT INTO tara VALUES (51, 'Egypt', 10);
INSERT INTO tara VALUES (52, 'El Salvador', 16);
INSERT INTO tara VALUES (53, 'Equatorial Guinea', 12);
INSERT INTO tara VALUES (54, 'Eritrea', 11);
INSERT INTO tara VALUES (55, 'Estonia', 2);
INSERT INTO tara VALUES (56, 'Eswatini', 13);
INSERT INTO tara VALUES (57, 'Ethiopia', 11);
INSERT INTO tara VALUES (58, 'Fiji', 20);
INSERT INTO tara VALUES (59, 'Finland', 2);
INSERT INTO tara VALUES (60, 'France', 4);
INSERT INTO tara VALUES (61, 'Gabon', 12);
INSERT INTO tara VALUES (62, 'Gambia', 14);
INSERT INTO tara VALUES (63, 'Georgia', 9);
INSERT INTO tara VALUES (64, 'Germany', 4);
INSERT INTO tara VALUES (65, 'Ghana', 14);
INSERT INTO tara VALUES (66, 'Greece', 3);
INSERT INTO tara VALUES (67, 'Grenada', 15);
INSERT INTO tara VALUES (68, 'Guatemala', 16);
INSERT INTO tara VALUES (69, 'Guinea', 14);
INSERT INTO tara VALUES (70, 'Guinea-Bissau', 14);
INSERT INTO tara VALUES (71, 'Guyana', 17);
INSERT INTO tara VALUES (72, 'Haiti', 15);
INSERT INTO tara VALUES (73, 'Honduras', 16);
INSERT INTO tara VALUES (74, 'Hungary', 1);
INSERT INTO tara VALUES (75, 'Iceland', 2);
INSERT INTO tara VALUES (76, 'India', 8);
INSERT INTO tara VALUES (77, 'Indonesia', 7);
INSERT INTO tara VALUES (78, 'Iran', 8);
INSERT INTO tara VALUES (79, 'Iraq', 9);
INSERT INTO tara VALUES (80, 'Ireland', 2);
INSERT INTO tara VALUES (81, 'Israel', 9);
INSERT INTO tara VALUES (82, 'Italy', 3);
INSERT INTO tara VALUES (83, 'Ivory Coast', 14);
INSERT INTO tara VALUES (84, 'Jamaica', 15);
INSERT INTO tara VALUES (85, 'Japan', 6);
INSERT INTO tara VALUES (86, 'Jordan', 9);
INSERT INTO tara VALUES (87, 'Kazakhstan', 5);
INSERT INTO tara VALUES (88, 'Kenya', 11);
INSERT INTO tara VALUES (89, 'Kiribati', 21);
INSERT INTO tara VALUES (90, 'North Korea', 6);
INSERT INTO tara VALUES (91, 'South Korea', 6);
INSERT INTO tara VALUES (92, 'Kosovo', 3);
INSERT INTO tara VALUES (93, 'Kuwait', 9);
INSERT INTO tara VALUES (94, 'Kyrgyzstan', 5);
INSERT INTO tara VALUES (95, 'Laos', 7);
INSERT INTO tara VALUES (96, 'Latvia', 2);
INSERT INTO tara VALUES (97, 'Lebanon', 9);
INSERT INTO tara VALUES (98, 'Lesotho', 13);
INSERT INTO tara VALUES (99, 'Liberia', 14);
INSERT INTO tara VALUES (100, 'Libya', 10);
INSERT INTO tara VALUES (101, 'Liechtenstein', 4);
INSERT INTO tara VALUES (102, 'Lithuania', 2);
INSERT INTO tara VALUES (103, 'Luxembourg', 4);
INSERT INTO tara VALUES (104, 'Madagascar', 11);
INSERT INTO tara VALUES (105, 'Malawi', 11);
INSERT INTO tara VALUES (106, 'Malaysia', 7);
INSERT INTO tara VALUES (107, 'Maldives', 8);
INSERT INTO tara VALUES (108, 'Mali', 14);
INSERT INTO tara VALUES (109, 'Malta', 3);
INSERT INTO tara VALUES (110, 'Marshall Islands', 21);
INSERT INTO tara VALUES (111, 'Mauritania', 14);
INSERT INTO tara VALUES (112, 'Mauritius', 11);
INSERT INTO tara VALUES (113, 'Mexico', 16);
INSERT INTO tara VALUES (114, 'Micronesia', 21);
INSERT INTO tara VALUES (115, 'Moldova', 1);
INSERT INTO tara VALUES (116, 'Monaco', 4);
INSERT INTO tara VALUES (117, 'Mongolia', 6);
INSERT INTO tara VALUES (118, 'Montenegro', 3);
INSERT INTO tara VALUES (119, 'Morocco', 10);
INSERT INTO tara VALUES (120, 'Mozambique', 11);
INSERT INTO tara VALUES (121, 'Myanmar', 7);
INSERT INTO tara VALUES (122, 'Namibia', 13);
INSERT INTO tara VALUES (123, 'Nauru', 21);
INSERT INTO tara VALUES (124, 'Nepal', 8);
INSERT INTO tara VALUES (125, 'Netherlands', 4);
INSERT INTO tara VALUES (126, 'New Zealand', 19);
INSERT INTO tara VALUES (127, 'Nicaragua', 16);
INSERT INTO tara VALUES (128, 'Niger', 14);
INSERT INTO tara VALUES (129, 'Nigeria', 14);
INSERT INTO tara VALUES (130, 'North Macedonia', 3);
INSERT INTO tara VALUES (131, 'Norway', 2);
INSERT INTO tara VALUES (132, 'Oman', 9);
INSERT INTO tara VALUES (133, 'Pakistan', 8);
INSERT INTO tara VALUES (134, 'Palau', 21);
INSERT INTO tara VALUES (135, 'Panama', 16);
INSERT INTO tara VALUES (136, 'Papua New Guinea', 20);
INSERT INTO tara VALUES (137, 'Paraguay', 17);
INSERT INTO tara VALUES (138, 'Peru', 17);
INSERT INTO tara VALUES (139, 'Philippines', 7);
INSERT INTO tara VALUES (140, 'Poland', 1);
INSERT INTO tara VALUES (141, 'Portugal', 3);
INSERT INTO tara VALUES (142, 'Qatar', 9);
INSERT INTO tara VALUES (143, 'Romania', 1);
INSERT INTO tara VALUES (144, 'Russia', 1);
INSERT INTO tara VALUES (145, 'Rwanda', 11);
INSERT INTO tara VALUES (146, 'Saint Kitts and Nevis', 15);
INSERT INTO tara VALUES (147, 'Saint Lucia', 15);
INSERT INTO tara VALUES (148, 'Saint Vincent and the Grenadines', 15);
INSERT INTO tara VALUES (149, 'Samoa', 22);
INSERT INTO tara VALUES (150, 'San Marino', 3);
INSERT INTO tara VALUES (151, 'Sao Tome and Principe', 12);
INSERT INTO tara VALUES (152, 'Saudi Arabia', 9);
INSERT INTO tara VALUES (153, 'Senegal', 14);
INSERT INTO tara VALUES (154, 'Serbia', 3);
INSERT INTO tara VALUES (155, 'Seychelles', 11);
INSERT INTO tara VALUES (156, 'Sierra Leone', 14);
INSERT INTO tara VALUES (157, 'Singapore', 7);
INSERT INTO tara VALUES (158, 'Slovakia', 1);
INSERT INTO tara VALUES (159, 'Slovenia', 3);
INSERT INTO tara VALUES (160, 'Solomon Islands', 20);
INSERT INTO tara VALUES (161, 'Somalia', 11);
INSERT INTO tara VALUES (162, 'South Africa', 13);
INSERT INTO tara VALUES (163, 'South Sudan', 11);
INSERT INTO tara VALUES (164, 'Spain', 3);
INSERT INTO tara VALUES (165, 'Sri Lanka', 8);
INSERT INTO tara VALUES (166, 'Sudan', 10);
INSERT INTO tara VALUES (167, 'Suriname', 17);
INSERT INTO tara VALUES (168, 'Sweden', 2);
INSERT INTO tara VALUES (169, 'Switzerland', 4);
INSERT INTO tara VALUES (170, 'Syria', 9);
INSERT INTO tara VALUES (171, 'Taiwan', 6);
INSERT INTO tara VALUES (172, 'Tajikistan', 5);
INSERT INTO tara VALUES (173, 'Tanzania', 11);
INSERT INTO tara VALUES (174, 'Thailand', 7);
INSERT INTO tara VALUES (175, 'Togo', 14);
INSERT INTO tara VALUES (176, 'Tonga', 22);
INSERT INTO tara VALUES (177, 'Trinidad and Tobago', 15);
INSERT INTO tara VALUES (178, 'Tunisia', 10);
INSERT INTO tara VALUES (179, 'Turkey', 9);
INSERT INTO tara VALUES (180, 'Turkmenistan', 5);
INSERT INTO tara VALUES (181, 'Tuvalu', 22);
INSERT INTO tara VALUES (182, 'Uganda', 11);
INSERT INTO tara VALUES (183, 'Ukraine', 1);
INSERT INTO tara VALUES (184, 'United Arab Emirates', 9);
INSERT INTO tara VALUES (185, 'United Kingdom', 2);
INSERT INTO tara VALUES (186, 'United States', 18);
INSERT INTO tara VALUES (187, 'Uruguay', 17);
INSERT INTO tara VALUES (188, 'Uzbekistan', 5);
INSERT INTO tara VALUES (189, 'Vanuatu', 20);
INSERT INTO tara VALUES (190, 'Vatican City', 3);
INSERT INTO tara VALUES (191, 'Venezuela', 17);
INSERT INTO tara VALUES (192, 'Vietnam', 7);
INSERT INTO tara VALUES (193, 'Yemen', 9);
INSERT INTO tara VALUES (194, 'Zambia', 11);
INSERT INTO tara VALUES (195, 'Zimbabwe', 11);

SELECT * FROM tara;

INSERT INTO arbitru VALUES(1, 'Lahyani', 'Mohamed', 168);
INSERT INTO arbitru VALUES(2, 'Veljovic', 'Marijana', 154);
INSERT INTO arbitru VALUES(3, 'Tourte', 'Aurelie', 60);
INSERT INTO arbitru VALUES(4, 'Bernardes', 'Carlos', 24);
INSERT INTO arbitru VALUES(5, 'Nour', 'Adel', 51);
INSERT INTO arbitru VALUES(6, 'Asderaki-Moore', 'Eva', 66);
INSERT INTO arbitru VALUES(7, 'Dumusois', 'Damien', 60);

SELECT * FROM arbitru;

INSERT INTO jucator VALUES(100, 'Karatsev', 'Aslan', to_date('04-09-1993', 'DD-MM-YYYY'), 144, 20, 185, 85, 'DREAPTA', 2013, null, null, 'Yahor Yatsyk');
INSERT INTO jucator VALUES(101, 'Ruud', 'Casper', to_date('22-12-1998', 'DD-MM-YYYY'), 131, 8, 183, 77, 'DREAPTA', 2015, null, 7, 'Christian Ruud');
INSERT INTO jucator VALUES(102, 'Harris', 'Lloyd', to_date('24-02-1997', 'DD-MM-YYYY'), 162, 32, 193, 80, 'DREAPTA', 2015, null, null, 'Anthony Harris');
INSERT INTO jucator VALUES(103, 'Thiem', 'Dominic', to_date('03-09-1993', 'DD-MM-YYYY'), 10, 15, 185, 79, 'DREAPTA', 2011, null, null, 'Nicolas Massu');
INSERT INTO jucator VALUES(104, 'Garin', 'Cristian', to_date('30-05-1996', 'DD-MM-YYYY'), 35, 18, 185, 85, 'DREAPTA', 2011, null, null, 'Franco Davin');
INSERT INTO jucator VALUES(105, 'Sinner', 'Jannik', to_date('16-08-2001', 'DD-MM-YYYY'), 82, 11, 185, 68, 'DREAPTA', 2018, null, null, 'Riccardo Piatti, Andrea Volpini');
INSERT INTO jucator VALUES(106, 'Zverev', 'Alexander', to_date('20-04-1997', 'DD-MM-YYYY'), 64, 3, 198, 90, 'DREAPTA', 2013, null, 3, 'Alexander Zverev Sr.');
INSERT INTO jucator VALUES(107, 'Berrettini', 'Matteo', to_date('12-04-1996', 'DD-MM-YYYY'), 82, 7, 196, 90, 'DREAPTA', 2015, null, 6, 'Vincenzo Santopadre, Marco Gulisano, Umberto Rianna');
INSERT INTO jucator VALUES(108, 'Schwartzman', 'Diego', to_date('16-08-1992', 'DD-MM-YYYY'), 7, 13, 170, 64, 'DREAPTA', 2010, null, null, 'Juan Ignacio Chela, Alejandro Fabbri');
INSERT INTO jucator VALUES(109, 'Monfils', 'Gael', to_date('01-09-1986', 'DD-MM-YYYY'), 60, 19, 193, 85, 'DREAPTA', 2004, null, null, 'Gunter Bresnik, Richard Ruckelshausen');
INSERT INTO jucator VALUES(110, 'Carreno Busta', 'Pablo', to_date('12-07-1991', 'DD-MM-YYYY'), 164, 21, 188, 78, 'DREAPTA', 2009, null, null, 'Samuel Lopez, Cesar Fabregas');
INSERT INTO jucator VALUES(111, 'Humbert', 'Ugo', to_date('26-06-1998', 'DD-MM-YYYY'), 60, 31, 188, 72, 'STANGA', 2016, null, null, 'Nicolas Copin, Thierry Ascione');
INSERT INTO jucator VALUES(112, 'Medvedev', 'Daniil', to_date('11-02-1996', 'DD-MM-YYYY'), 144, 2, 198, 83, 'DREAPTA', 2014, null, 2, 'Gilles Cervara');
INSERT INTO jucator VALUES(113, 'Auger-Aliassime', 'Felix', to_date('08-08-2000', 'DD-MM-YYYY'), 32, 9, 193, 88, 'DREAPTA', 2017, null, 8, 'Frederic Fontang, Toni Nadal');
INSERT INTO jucator VALUES(114, 'Basilashvili', 'Nikoloz', to_date('23-02-1992', 'DD-MM-YYYY'), 63, 23, 185, 79, 'DREAPTA', 2008, null, null, null);
INSERT INTO jucator VALUES(115, 'Evans', 'Daniel', to_date('23-05-1990', 'DD-MM-YYYY'), 185, 26, 175, 75, 'DREAPTA', 2006, null, null, 'Sebastian Prieto');
INSERT INTO jucator VALUES(116, 'Rublev', 'Andrey', to_date('20-10-1997', 'DD-MM-YYYY'), 144, 5, 188, 70, 'DREAPTA', 2014, null, 5, 'Fernando Vicente');
INSERT INTO jucator VALUES(117, 'Sonego', 'Lorenzo', to_date('11-05-1995', 'DD-MM-YYYY'), 82, 27, 191, 76, 'DREAPTA', 2013, null, null, 'Gipo Arbino');
INSERT INTO jucator VALUES(118, 'Norrie', 'Cameron', to_date('23-08-1995', 'DD-MM-YYYY'), 185, 12, 188, 82, 'STANGA', 2016, null, null, 'Facundo Lugones');
INSERT INTO jucator VALUES(119, 'Shapovalov', 'Denis', to_date('15-04-1999', 'DD-MM-YYYY'), 32, 14, 183, 75, 'STANGA', 2017, null, null, 'Tessa Shapovalova, Mikhail Youzhny, Jamie Delgado');
INSERT INTO jucator VALUES(120, 'Dimitrov', 'Grigor', to_date('16-05-1991', 'DD-MM-YYYY'), 26, 28, 191, 81, 'DREAPTA', 2008, null, null, 'Dante Bottini');
INSERT INTO jucator VALUES(121, 'Cilic', 'Marin', to_date('29-09-1988', 'DD-MM-YYYY'), 41, 29, 198, 89, 'DREAPTA', 2005, null, null, 'Vedran Martic, Vilim Visak');
INSERT INTO jucator VALUES(122, 'Khachanov', 'Karen', to_date('21-05-1996', 'DD-MM-YYYY'), 144, 30, 198, 87, 'DREAPTA', 2013, null, null, 'Jose Clavet');
INSERT INTO jucator VALUES(123, 'Alcaraz', 'Carlos', to_date('05-05-2003', 'DD-MM-YYYY'), 164, 33, 185, 72, 'DREAPTA', 2018, null, null, 'Juan Carlos Ferrero');
INSERT INTO jucator VALUES(124, 'Djokovic', 'Novak', to_date('22-05-1987', 'DD-MM-YYYY'), 154, 1, 188, 77, 'DREAPTA', 2003, null, 1, 'Marian Vajda, Goran Ivanisevic');
INSERT INTO jucator VALUES(125, 'Isner', 'John', to_date('26-04-1985', 'DD-MM-YYYY'), 186, 24, 208, 108, 'DREAPTA', 2007, null, null, 'David Macpherson');
INSERT INTO jucator VALUES(126, 'Bautista Agut', 'Roberto', to_date('14-04-1988', 'DD-MM-YYYY'), 164, 17, 183, 75, 'DREAPTA', 2005, null, null, 'Daniel Gimeno-Traver, Tomas Carbonell');
INSERT INTO jucator VALUES(127, 'Murray', 'Andy', to_date('15-05-1987', 'DD-MM-YYYY'), 185, 87, 191, 82, 'DREAPTA', 2005, null, null, 'Daniel Vallverdu');
INSERT INTO jucator VALUES(128, 'Tsitsipas', 'Stefanos', to_date('12-08-1998', 'DD-MM-YYYY'), 66, 4, 193, 85, 'DREAPTA', 2016, null, 4, 'Apostolos Tsitsipas');
INSERT INTO jucator VALUES(129, 'Fritz', 'Taylor', to_date('28-10-1997', 'DD-MM-YYYY'), 186, 22, 193, 86, 'DREAPTA', 2015, null, null, 'Michael Russell, Paul Annacone, David Nainkin');
INSERT INTO jucator VALUES(130, 'Hurkacz', 'Hubert', to_date('11-02-1997', 'DD-MM-YYYY'), 140, 10, 196, 81, 'DREAPTA', 2015, null, null, 'Craig Boynton');
INSERT INTO jucator VALUES(131, 'Opelka', 'Reilly', to_date('28-08-1997', 'DD-MM-YYYY'), 186, 25, 211, 102, 'DREAPTA', 2015, null, null, 'Jay Berger, Jean-Yves Aubone');

SELECT * FROM jucator;

INSERT INTO meci VALUES(500, 124, 117, 1, 1, 1, to_date('30-10-2023 18:00', 'DD-MM-YYYY HH24:MI'), 124, '6-3 6-4', 93, 10200);
INSERT INTO meci VALUES(501, 125, 129, 3, 5, 1, to_date('30-10-2023 14:00', 'DD-MM-YYYY HH24:MI'), 129, '7-5 6-7 7-6', 150, 200);
INSERT INTO meci VALUES(502, 105, 123, 2, 2, 1, to_date('30-10-2023 18:00', 'DD-MM-YYYY HH24:MI'), 105, '2-6 6-3 6-3', 128, 1500);
INSERT INTO meci VALUES(503, 121, 107, 1, 1, 1, to_date('30-10-2023 12:00', 'DD-MM-YYYY HH24:MI'), 107, '7-5 7-5', 106, 10200);
INSERT INTO meci VALUES(504, 101, 104, 2, 3, 1, to_date('30-10-2023 15:00', 'DD-MM-YYYY HH24:MI'), 101, '6-4 6-4', 97, 1500);
INSERT INTO meci VALUES(505, 109, 126, 3, 7, 1, to_date('30-10-2023 17:00', 'DD-MM-YYYY HH24:MI'), 126, '4-6 6-3 7-5', 133, 200);
INSERT INTO meci VALUES(506, 103, 118, 2, 2, 1, to_date('30-10-2023 12:00', 'DD-MM-YYYY HH24:MI'), 103, '6-4 2-6 6-3', 114, 1500);
INSERT INTO meci VALUES(507, 122, 106, 1, 4, 1, to_date('30-10-2023 15:00', 'DD-MM-YYYY HH24:MI'), 106, '6-4 6-3', 99, 10200);
INSERT INTO meci VALUES(508, 116, 130, 1, 6, 1, to_date('31-10-2023 12:00', 'DD-MM-YYYY HH24:MI'), 116, '7-6 6-4', 107, 10200);
INSERT INTO meci VALUES(509, 110, 112, 1, 4, 1, to_date('31-10-2023 15:00', 'DD-MM-YYYY HH24:MI'), 112, '7-5 6-3', 111, 10200);
INSERT INTO meci VALUES(510, 128, 127, 1, 7, 1, to_date('31-10-2023 18:00', 'DD-MM-YYYY HH24:MI'), 128, '6-3 6-3', 87, 10200);
INSERT INTO meci VALUES(511, 120, 115, 2, 7, 1, to_date('31-10-2023 12:00', 'DD-MM-YYYY HH24:MI'), 120, '5-7 6-3 7-5', 132, 1500);
INSERT INTO meci VALUES(512, 119, 111, 2, 6, 1, to_date('31-10-2023 15:00', 'DD-MM-YYYY HH24:MI'), 119, '6-2 6-2', 68, 1500);
INSERT INTO meci VALUES(513, 102, 113, 2, 1, 1, to_date('31-10-2023 18:00', 'DD-MM-YYYY HH24:MI'), 113, '6-3 6-2', 72, 1500);
INSERT INTO meci VALUES(514, 100, 131, 3, 5, 1, to_date('31-10-2023 14:00', 'DD-MM-YYYY HH24:MI'), 100, '7-6 4-6 7-6', 139, 200);
INSERT INTO meci VALUES(515, 108, 114, 3, 3, 1, to_date('31-10-2023 17:00', 'DD-MM-YYYY HH24:MI'), 108, '6-4 7-5', 88, 200);
INSERT INTO meci VALUES(516, 124, 129, 1, 4, 2, to_date('01-11-2023 18:00', 'DD-MM-YYYY HH24:MI'), 124, '6-4 6-4', 92, 10200);
INSERT INTO meci VALUES(517, 105, 107, 1, 1, 2, to_date('01-11-2023 12:00', 'DD-MM-YYYY HH24:MI'), 105, '6-3 4-6 7-5', 119, 10200);
INSERT INTO meci VALUES(518, 103, 106, 1, 2, 2, to_date('01-11-2023 15:00', 'DD-MM-YYYY HH24:MI'), 106, '6-4 6-3', 85, 10200);
INSERT INTO meci VALUES(519, 101, 126, 2, 3, 2, to_date('01-11-2023 16:30', 'DD-MM-YYYY HH24:MI'), 101, '7-5 6-4', 101, 1500);
INSERT INTO meci VALUES(520, 128, 119, 1, 2, 2, to_date('02-11-2023 15:00', 'DD-MM-YYYY HH24:MI'), 128, '6-3 7-5', 123, 10186);
INSERT INTO meci VALUES(521, 120, 113, 2, 3, 2, to_date('02-11-2023 16:30', 'DD-MM-YYYY HH24:MI'), 120, '2-6 6-4 6-4', 144, 1153);
INSERT INTO meci VALUES(522, 116, 100, 1, 5, 2, to_date('02-11-2023 12:00', 'DD-MM-YYYY HH24:MI'), 116, '7-6 6-3', 111, 10162);
INSERT INTO meci VALUES(523, 108, 112, 1, 4, 2, to_date('02-11-2023 18:00', 'DD-MM-YYYY HH24:MI'), 112, '6-4 6-1', 86, 10187);
INSERT INTO meci VALUES(524, 128, 120, 1, 1, 3, to_date('03-11-2023 12:00', 'DD-MM-YYYY HH24:MI'), 120, '6-3 7-5', 90, 10189);
INSERT INTO meci VALUES(525, 116, 112, 1, 2, 3, to_date('03-11-2023 15:00', 'DD-MM-YYYY HH24:MI'), 116, '6-7 7-5 7-5', 170, 10190);
INSERT INTO meci VALUES(526, 124, 105, 1, 3, 3, to_date('03-11-2023 18:00', 'DD-MM-YYYY HH24:MI'), 124, '6-3 7-6', 101, 10190);
INSERT INTO meci VALUES(527, 101, 106, 2, 4, 3, to_date('03-11-2023 16:30', 'DD-MM-YYYY HH24:MI'), 101, '6-4 4-6 6-4', 139, 1489);
INSERT INTO meci VALUES(528, 120, 116, 1, 3, 4, to_date('04-11-2023 17:00', 'DD-MM-YYYY HH24:MI'), 116, '6-4 7-5', 98, 10194);
INSERT INTO meci VALUES(529, 124, 101, 1, 1, 4, to_date('04-11-2023 14:00', 'DD-MM-YYYY HH24:MI'), 124, '6-1 6-2', 62, 10194);
INSERT INTO meci VALUES(530, 124, 116, 1, 3, 5, to_date('05-11-2023 16:30', 'DD-MM-YYYY HH24:MI'), 124, '7-5 6-3', 115, 10200);

SELECT * FROM meci;

CREATE OR REPLACE FUNCTION HASH_PASSWORD
    (user_name in varchar2,
     parola  in varchar2)
RETURN varchar2
IS
    hash_parola varchar2(255);
    input_string varchar2(255);
    salt varchar2(255) := '2GSK5LS73NC8SVB9DTQ6ZK810PXLGWZUJSA';
BEGIN
    input_string := parola || substr(salt, 10, 20) || user_name || substr(salt, 25, 30);
    hash_parola := dbms_crypto.hash(
        src => utl_raw.cast_to_raw(input_string),
        typ => dbms_crypto.hash_sh256
    );
    return hash_parola;
END HASH_PASSWORD;
/

DECLARE
    username user_.user_name%TYPE;
    tara_id user_.id_tara%TYPE;
    parola VARCHAR2(10) := 'Parola1!';
    parola_hash VARCHAR2(255);
    data_nasterii user_.data_nastere%TYPE;
    email user_.email%TYPE;
BEGIN
    FOR i IN 1..150 LOOP
        username := dbms_random.string('U', dbms_random.value(7, 15)) || i;
        parola_hash := hash_password(username, parola);
        email := username || '@gmail.com';
        
        SELECT id_tara INTO tara_id
        FROM (SELECT id_tara FROM tara
              ORDER BY dbms_random.value)
        WHERE rownum = 1;
        
        data_nasterii := TO_DATE(TRUNC(dbms_random.value(to_char(DATE '1960-01-01', 'J'),
                                                         to_char(DATE '2008-01-01', 'J'))), 'J');
                                                         
        INSERT INTO user_
        VALUES (i, username, parola_hash, email, data_nasterii, 0, tara_id);
    END LOOP;
END;
/
                                                         
select * FROM user_;  

DECLARE
    meci_id bilet.id_meci%TYPE;
    data_ora_meci meci.data_ora%TYPE;
    pret_intreg_meci tur.pret%TYPE;
    user_id bilet.id_user%TYPE;
    cant bilet.cantitate%TYPE;
    comanda bilet.nr_comanda%TYPE;
    data_cumparare bilet.data_achizitie%TYPE;
    pret_platit bilet.pret_platit%TYPE;
BEGIN
    FOR i IN 1..150000 LOOP
        SELECT id_user INTO user_id
        FROM (SELECT id_user FROM user_
              ORDER BY dbms_random.value)
        WHERE rownum = 1;
        
        SELECT id_meci, data_ora, pret 
        INTO meci_id, data_ora_meci, pret_intreg_meci 
        FROM (SELECT id_meci, data_ora, pret 
              FROM meci m, tur t
              WHERE m.id_tur = t.id_tur
              ORDER BY dbms_random.value)
        WHERE rownum = 1;
        
        data_cumparare := TO_DATE(TRUNC(dbms_random.value(to_char(DATE '2023-01-01', 'J'),
                                                          to_char(data_ora_meci, 'J'))), 'J');
        cant := trunc(dbms_random.value(1, 6));
        comanda := trunc(dbms_random.value(1000000000, 9999999999));
        
        IF dbms_random.value(0, 1) < 0.3 THEN
            pret_platit := round(pret_intreg_meci * 0.8);
        ELSE
            pret_platit := pret_intreg_meci;
        END IF;
        
        INSERT INTO bilet 
        VALUES (i, meci_id, user_id, cant, comanda, pret_platit, data_cumparare); 
    END LOOP;
END;
/

SELECT * FROM bilet ORDER BY id_bilet DESC;

commit;

-- acordare de privilegii SELECT pentru userul OLAP
BEGIN
    FOR i IN (SELECT table_name
              FROM user_tables
              WHERE lower(table_name) IN ('regiune', 'subregiune',
                                          'tara', 'tur', 'teren',
                                          'arbitru', 'jucator',
                                          'meci', 'bilet', 'user_')) LOOP
        EXECUTE IMMEDIATE 'GRANT SELECT ON ' || i.table_name || ' TO dwbi_olap';
    END LOOP;
END;
/

-----------------------------------------------------------------------------------------------------------------------------------------

-- logat ca dwbi_olap

-- 3. Crearea bazei de date depozit

CREATE TABLE regiune(
    id_regiune NUMBER,
    tara VARCHAR2(50) NOT NULL,
    subregiune VARCHAR2(50) NOT NULL,
    regiune VARCHAR2(50) NOT NULL
);

CREATE TABLE user_(
    id_user NUMBER,
    user_name VARCHAR2(255) NOT NULL,
    email VARCHAR2(255) NOT NULL,
    data_nastere DATE NOT NULL
);

CREATE TABLE timp(
    id_timp NUMBER,
    data DATE NOT NULL,
    sapt_luna_an VARCHAR2(10) NOT NULL,
    semestru_an VARCHAR2(10) NOT NULL,
    luna_an VARCHAR2(10) NOT NULL,
    an NUMBER NOT NULL,
    zi NUMBER NOT NULL,
    luna NUMBER NOT NULL,
    nume_luna VARCHAR2(20) NOT NULL
);

CREATE TABLE meci(
    id_meci NUMBER,
    data_ora DATE NOT NULL,
    scor VARCHAR2(20),
    nume_juc1 VARCHAR2(25),
    prenume_juc1 VARCHAR2(25),
    nume_juc2 VARCHAR2(25),
    prenume_juc2 VARCHAR2(25),
    clasament_juc1 NUMBER(4),
    clasament_juc2 NUMBER(4),
    tara_juc1 VARCHAR2(50),
    tara_juc2 VARCHAR2(50),
    tur VARCHAR2(20) NOT NULL,
    teren VARCHAR2(20) NOT NULL,
    nume_arbitru VARCHAR2(25),
    prenume_arbitru VARCHAR2(25),
    pret NUMBER NOT NULL
);

CREATE TABLE bilet(
    id_bilet NUMBER,
    id_meci NUMBER NOT NULL,
    id_user NUMBER NOT NULL,
    id_regiune NUMBER NOT NULL,
    id_timp NUMBER NOT NULL,
    nr_comanda NUMBER NOT NULL,
    cantitate NUMBER NOT NULL,
    pret_unitar NUMBER NOT NULL,
    valoare NUMBER NOT NULL
);

-- 4. Popularea cu informatii a bazei de date depozit folosind ca sursa
--    datele din baza de date OLTP

INSERT INTO regiune
SELECT t.id_tara, t.nume, s.nume, r.nume
FROM dwbi_oltp.tara t, dwbi_oltp.subregiune s, dwbi_oltp.regiune r
WHERE t.id_subregiune = s.id_subregiune
AND s.id_regiune = r.id_regiune;

SELECT * FROM regiune;

INSERT INTO user_
SELECT id_user, user_name, email, data_nastere
FROM dwbi_oltp.user_;

SELECT * FROM user_;

INSERT INTO timp
SELECT to_char(data, 'j') id_timp, data data, 
       to_char(data, 'yyyy-mm-w') sapt_luna_an,
       CASE
           WHEN EXTRACT(MONTH FROM data) < 7 THEN to_char(data, 'yyyy-') || '1'
           ELSE to_char(data, 'yyyy-') || '2'
       END semestru_an,
       to_char(data, 'yyyy-mm') luna_an,
       EXTRACT(YEAR FROM data) an,
       EXTRACT(DAY FROM data) zi,
       EXTRACT(MONTH FROM data) luna,
       to_char(data, 'Month') nume_luna
FROM (SELECT trunc(to_date('01-01-2023', 'DD-MM-YYYY') +rownum-1) data
      FROM dual
      CONNECT BY rownum <= 365);

SELECT * FROM timp;

INSERT INTO meci
SELECT id_meci, data_ora, scor, j1.nume, j1.prenume, j2.nume, j2.prenume,
       j1.clasament, j2.clasament, t1.nume, t2.nume, tu.denumire, 
       te.denumire, a.nume, a.prenume, tu.pret
FROM dwbi_oltp.meci m, dwbi_oltp.jucator j1, dwbi_oltp.jucator j2,
     dwbi_oltp.tara t1, dwbi_oltp.tara t2, dwbi_oltp.tur tu,
     dwbi_oltp.teren te, dwbi_oltp.arbitru a
WHERE m.id_jucator1 = j1.id_jucator
AND m.id_jucator2 = j2.id_jucator
AND j1.id_tara = t1.id_tara
AND j2.id_tara = t2.id_tara
AND m.id_tur = tu.id_tur
AND m.id_teren = te.id_teren
AND m.id_arbitru = a.id_arbitru;

SELECT * FROM meci;

INSERT INTO bilet
SELECT id_bilet, id_meci, b.id_user, u.id_tara, 
       to_char(data_achizitie, 'j'), nr_comanda, 
       cantitate, pret_platit, round(cantitate * pret_platit, 2)
FROM dwbi_oltp.bilet b, dwbi_oltp.user_ u
WHERE b.id_user = u.id_user;

SELECT * FROM bilet ORDER BY id_bilet DESC;

commit;

-- 5. Definirea constrangerilor

-- chei primare definite implicit cu ENABLE VALIDATE si optiunea RELY
ALTER TABLE regiune
ADD CONSTRAINT pk_regiune 
PRIMARY KEY (id_regiune) RELY;

ALTER TABLE user_
ADD CONSTRAINT pk_user 
PRIMARY KEY (id_user) RELY;

ALTER TABLE timp
ADD CONSTRAINT pk_timp 
PRIMARY KEY (id_timp) RELY;

ALTER TABLE meci
ADD CONSTRAINT pk_meci 
PRIMARY KEY (id_meci) RELY;

-- cheie primara definita cu RELY DISABLE NOVALIDATE
ALTER TABLE bilet
ADD CONSTRAINT pk_bilet PRIMARY KEY (id_bilet)
RELY DISABLE NOVALIDATE;

-- chei externe definite cu RELY DISABLE NOVALIDATE
ALTER TABLE bilet
ADD CONSTRAINT fk_bilet_meci FOREIGN KEY (id_meci)
REFERENCES meci(id_meci)
RELY DISABLE NOVALIDATE;

ALTER TABLE bilet
ADD CONSTRAINT fk_bilet_user FOREIGN KEY (id_user)
REFERENCES user_(id_user)
RELY DISABLE NOVALIDATE;

ALTER TABLE bilet
ADD CONSTRAINT fk_bilet_regiune FOREIGN KEY (id_regiune)
REFERENCES regiune(id_regiune)
RELY DISABLE NOVALIDATE;

ALTER TABLE bilet
ADD CONSTRAINT fk_bilet_timp FOREIGN KEY (id_timp)
REFERENCES timp(id_timp)
RELY DISABLE NOVALIDATE;

-- 6. Definirea indecsilor si a cererilor SQL insotite de planul
--    de executie al acestora (din care sa reiasa ca optimizatorul
--    utilizeaza eficient indecsii definiti)

CREATE BITMAP INDEX bmp_meci_tur
ON meci(tur);

ANALYZE INDEX bmp_meci_tur COMPUTE STATISTICS;

-- Obtineti numarul de meciuri de 3 seturi din turul secund 
-- si procentajul acestora din numarul total de meciuri

SELECT nr_3_tur2 nr_3seturi_tur2,
       ROUND(nr_3_tur2 / (SELECT COUNT(*)
                          FROM meci) * 100, 2) || '%' procent
FROM
(SELECT COUNT(*) nr_3_tur2
 FROM meci
 WHERE tur = '2nd Round'
 AND 3 = (SELECT COUNT(*)
          FROM dual
          CONNECT BY REGEXP_SUBSTR(scor, '[^ ]+', 1, level) IS NOT NULL));

EXPLAIN PLAN 
SET STATEMENT_ID = 'ex6' 
FOR 
SELECT nr_3_tur2 nr_3seturi_tur2,
       ROUND(nr_3_tur2 / (SELECT COUNT(*)
                          FROM meci) * 100, 2) || '%' procent
FROM
(SELECT COUNT(*) nr_3_tur2
 FROM meci
 WHERE tur = '2nd Round'
 AND 3 = (SELECT COUNT(*)
          FROM dual
          CONNECT BY REGEXP_SUBSTR(scor, '[^ ]+', 1, level) IS NOT NULL));

SELECT plan_table_output 
FROM 
table(dbms_xplan.display('plan_table', 'ex6','serial'));

-- 7. Definirea obiectelor de tip dimensiune, validarea acestora
--    (din care sa reiasa ca datele respecta constrangerile impuse
--    princ aceste tipuri de obiecte)

-- ierarhia an-semestru-luna-saptamana

CREATE DIMENSION timp_dim
LEVEL saptamana IS (timp.sapt_luna_an)
LEVEL luna IS (timp.luna_an)
LEVEL semestru IS (timp.semestru_an)
LEVEL an IS (timp.an)
HIERARCHY h1
(saptamana CHILD OF
 luna CHILD OF
 semestru CHILD OF
 an);
 
EXECUTE DBMS_DIMENSION.VALIDATE_DIMENSION(UPPER('timp_dim'), FALSE, TRUE, 'st_id');

SELECT * 
FROM DIMENSION_EXCEPTIONS
WHERE STATEMENT_ID = 'st_id';

-- 8. Definirea partitiilor, definirea cererilor SQL insotite de
--    planul de executie al acestora din care sa reiasa ca
--    optimizorul utilizeaza eficient partitiile

CREATE TABLE bilet_ord(
    id_bilet NUMBER,
    id_meci NUMBER NOT NULL,
    id_user NUMBER NOT NULL,
    id_regiune NUMBER NOT NULL,
    id_timp NUMBER NOT NULL,
    nr_comanda NUMBER NOT NULL,
    cantitate NUMBER NOT NULL,
    pret_unitar NUMBER NOT NULL,
    valoare NUMBER NOT NULL
)
PARTITION BY RANGE(id_timp)
(
 PARTITION bilete_sub2023 
 VALUES LESS THAN (2459946),
 -- 2459946 = TO_CHAR(TO_DATE('01/01/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_jan2023 
 VALUES LESS THAN (2459977),
 -- 2459977 = TO_CHAR(TO_DATE('01/02/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_feb2023
 VALUES LESS THAN (2460005),
 -- 2460005 = TO_CHAR(TO_DATE('01/03/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_mar2023 
 VALUES LESS THAN (2460036),
 -- 2460036 = TO_CHAR(TO_DATE('01/04/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_apr2023 
 VALUES LESS THAN (2460066),
 -- 2460066 = TO_CHAR(TO_DATE('01/05/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_mai2023 
 VALUES LESS THAN (2460097),
 -- 2460097 = TO_CHAR(TO_DATE('01/06/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_iun2023 
 VALUES LESS THAN (2460127),
 -- 2460127 = TO_CHAR(TO_DATE('01/07/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_iul2023 
 VALUES LESS THAN (2460158),
 -- 2460158 = TO_CHAR(TO_DATE('01/08/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_aug2023 
 VALUES LESS THAN (2460189),
 -- 2460189 = TO_CHAR(TO_DATE('01/09/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_sept2023 
 VALUES LESS THAN (2460219),
 -- 2460219 = TO_CHAR(TO_DATE('01/10/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_oct2023
 VALUES LESS THAN (2460250),
 -- 2460250 = TO_CHAR(TO_DATE('01/11/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_nov2023 
 VALUES LESS THAN (2460280),
 -- 2460280 = TO_CHAR(TO_DATE('01/12/2023','DD/MM/YYYY'),'J')
 PARTITION bilete_dec2023
 VALUES LESS THAN (2460311),
 -- 2460311 = TO_CHAR(TO_DATE('01/01/2024','DD/MM/YYYY'),'J')
 PARTITION bilete_rest
 VALUES LESS THAN (MAXVALUE)
);

INSERT INTO bilet_ord
SELECT * FROM bilet;

commit;

ANALYZE TABLE bilet_ord COMPUTE STATISTICS;

-- Afisati pentru fiecare meci numarul de bilete vandute
-- in luna septembrie 2023 si valoarea totala a acestora.

SELECT m.nume_juc1 || ' vs. ' || m.nume_juc2 meci, SUM(cantitate), SUM(valoare)
FROM bilet_ord b, meci m
WHERE id_timp BETWEEN TO_CHAR(TO_DATE('01/09/2023','DD/MM/YYYY'),'J')
                  AND TO_CHAR(TO_DATE('30/09/2023','DD/MM/YYYY'),'J')
AND b.id_meci = m.id_meci
GROUP BY m.nume_juc1, m.nume_juc2, m.data_ora
ORDER BY m.data_ora;

EXPLAIN PLAN 
SET STATEMENT_ID = 'ex8' 
FOR 
SELECT m.nume_juc1 || ' vs. ' || m.nume_juc2 meci, SUM(cantitate), SUM(valoare)
FROM bilet_ord b, meci m
WHERE id_timp BETWEEN TO_CHAR(TO_DATE('01/09/2023','DD/MM/YYYY'),'J')
                  AND TO_CHAR(TO_DATE('30/09/2023','DD/MM/YYYY'),'J')
AND b.id_meci = m.id_meci
GROUP BY m.nume_juc1, m.nume_juc2, m.data_ora
ORDER BY m.data_ora;

SELECT plan_table_output 
FROM 
table(dbms_xplan.display('plan_table', 'ex8','serial'));

-- 9. Optimizarea cererii SQL propusa in etapa de analiza

-- Afisati pentru fiecare tara lunile in care cantitatea 
-- de bilete vandute depaseste media anuala

WITH aux AS
(SELECT tara, luna, nume_luna, an, sum(cantitate) bilete_luna
FROM bilet b, regiune r, timp t
WHERE b.id_regiune = r.id_regiune
AND b.id_timp = t.id_timp
GROUP BY tara, nume_luna, luna, an)
SELECT tara, nume_luna, bilete_luna
FROM aux a
WHERE bilete_luna >= (SELECT sum(bilete_luna)/12
                      FROM aux
                      WHERE tara = a.tara
                      AND an = a.an)
ORDER BY tara, luna;

-- a. Planul de executie ales de optimizorul bazat pe cost

EXPLAIN PLAN 
SET STATEMENT_ID = 'ex9_neoptimizat' 
FOR 
WITH aux AS
(SELECT tara, luna, nume_luna, an, sum(cantitate) bilete_luna
FROM bilet b, regiune r, timp t
WHERE b.id_regiune = r.id_regiune
AND b.id_timp = t.id_timp
GROUP BY tara, nume_luna, luna, an)
SELECT tara, nume_luna, bilete_luna
FROM aux a
WHERE bilete_luna >= (SELECT sum(bilete_luna)/12
                      FROM aux
                      WHERE tara = a.tara
                      AND an = a.an)
ORDER BY tara, luna;

SELECT plan_table_output 
FROM table(dbms_xplan.display('plan_table', 'ex9_neoptimizat','serial'));

-- b. Sugestii de optimizare a cererii, specificand planul de executie obtinut

CREATE MATERIALIZED VIEW vm_bilete_timp_regiuni
BUILD IMMEDIATE
REFRESH COMPLETE
ON COMMIT
ENABLE QUERY REWRITE
AS 
SELECT tara, luna, nume_luna, an, sum(cantitate) bilete_luna
FROM bilet b, regiune r, timp t
WHERE b.id_regiune = r.id_regiune
AND b.id_timp = t.id_timp
GROUP BY tara, nume_luna, luna, an;

SELECT * FROM vm_bilete_timp_regiuni;

ANALYZE TABLE timp COMPUTE STATISTICS;
ANALYZE TABLE regiune COMPUTE STATISTICS;
ANALYZE TABLE bilet COMPUTE STATISTICS;
ANALYZE TABLE vm_bilete_timp_regiuni COMPUTE STATISTICS;

EXPLAIN PLAN 
SET STATEMENT_ID = 'ex9_optimizat' 
FOR 
WITH aux AS
(SELECT tara, luna, nume_luna, an, sum(cantitate) bilete_luna
FROM bilet b, regiune r, timp t
WHERE b.id_regiune = r.id_regiune
AND b.id_timp = t.id_timp
GROUP BY tara, nume_luna, luna, an)
SELECT tara, nume_luna, bilete_luna
FROM aux a
WHERE bilete_luna >= (SELECT sum(bilete_luna)/12
                      FROM aux
                      WHERE tara = a.tara
                      AND an = a.an)
ORDER BY tara, luna;

SELECT plan_table_output 
FROM table(dbms_xplan.display('plan_table', 'ex9_optimizat','serial'));

-- 10. Crearea rapoartelor cu complexitate diferita
 
-- 1. Afisati pentru fiecare tara top 3 meciuri pentru care utilizatorii au achizitionat bilete in anul 2023.
    
SELECT * FROM (
SELECT r.tara, m.nume_juc1, m.nume_juc2,
       SUM(b.cantitate) cantitate_totala,
       SUM(b.valoare) valoare_totala,
       DENSE_RANK() OVER (PARTITION BY r.tara
                          ORDER BY SUM(b.cantitate) DESC) d_rank_desc
FROM bilet b
JOIN meci m ON m.id_meci = b.id_meci
JOIN timp t ON t.id_timp = b.id_timp
JOIN regiune r ON r.id_regiune = b.id_regiune
WHERE t.an = 2023
GROUP BY r.tara, m.nume_juc1, m.nume_juc2)
WHERE d_rank_desc <= 3
ORDER BY 1;
 
-- 2. Afisati numarul de bilete achizitionate in semestrul 1 al anului 2023:
--    - pentru fiecare regiune, teren si tur
--    - pentru fiecare regiune si teren, indiferent de tur.
 
SELECT r.regiune, m.teren, m.tur,
       SUM(b.cantitate) AS total_bilete,
       GROUPING(r.regiune) AS g_regiune, GROUPING(m.teren) AS g_teren, 
       GROUPING(m.tur) AS g_tur, GROUPING_ID(r.regiune, m.teren, m.tur) AS gid
FROM bilet b
JOIN meci m ON m.id_meci = b.id_meci
JOIN timp t ON t.id_timp = b.id_timp
JOIN regiune r ON r.id_regiune = b.id_regiune
WHERE t.semestru_an = '2023-1'
GROUP BY CUBE (r.regiune, m.teren, m.tur)
HAVING GROUPING_ID(r.regiune, m.teren) = 0;
 
-- 3. Urmariti profitul de la luna la luna acumulat pentru fiecare meci in anul 2023.
 
SELECT id_meci, luna, total_valoare_meci, 
       total_valoare_meci - LAG(total_valoare_meci, 1)
           OVER (PARTITION BY id_meci ORDER BY luna) as dif
FROM (SELECT b.id_meci, t.luna, sum(b.valoare) total_valoare_meci
      FROM bilet b
      JOIN timp t ON t.id_timp = b.id_timp
      WHERE t.an = '2023'
      GROUP BY b.id_meci, t.luna)
ORDER BY 1, 2;
 
-- 4. Clasati tarile tinand cont de procentul de venit adus din totalul veniturilor in anul 2023.
 
SELECT r.tara,
       SUM(b.valoare) AS VANZARI,
       SUM(SUM(b.valoare)) OVER () AS TOTAL_VANZ,
       ROUND(RATIO_TO_REPORT(SUM(b.valoare)) OVER (), 6) AS RATIO_REP,
       DENSE_RANK() OVER (ORDER BY SUM(b.valoare) DESC) AS RANK
FROM bilet b
JOIN regiune r ON b.id_regiune = r.id_regiune 
GROUP BY r.tara;
 
-- 5.  Afisati media vanzarilor in ultimele 2 luni, luna curenta si luna urmatoare ale anului 2023 pentru fiecare meci.
 
SELECT  b.id_meci, t.luna, 
        SUM(b.valoare) valoare,
        ROUND(AVG(SUM(b.valoare))
            OVER (PARTITION BY b.id_meci 
                  ORDER BY b.id_meci, t.luna 
                  ROWS BETWEEN 2 PRECEDING AND 1 FOLLOWING), 2)  
            AS media_4_luni
FROM bilet b
JOIN timp t ON t.id_timp = b.id_timp
WHERE t.an = '2023'
GROUP BY b.id_meci, t.luna;


