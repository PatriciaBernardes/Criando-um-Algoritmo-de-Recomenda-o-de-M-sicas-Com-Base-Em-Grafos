# 🎧 Sistema de Recomendação de Músicas com Grafos (Neo4j & Cypher)

> **Desafio DIO - Bootcamp de Grafos para Ciência de Dados**

## 🎯 O Desafio: Construindo a Próxima Geração de Recomendações Musicais

Fui contratado para desenvolver um **sistema de recomendação de músicas** utilizando a tecnologia de **Grafos** com **Neo4j** e a linguagem **Cypher**. Em vez de depender de tabelas complexas, o objetivo foi mapear as interconexões entre usuários, músicas, artistas e gêneros para descobrir padrões de escuta e fornecer sugestões altamente personalizadas.

O uso do Neo4j permite explorar a **afinidade de usuários** (filtragem colaborativa) e a **similaridade de conteúdo musical** (filtragem baseada em atributos como `energy`, `danceability`, etc.) de forma nativa e eficiente.

---

## 🏗️ Modelagem do Grafo (Schema)

O modelo de dados é simples, mas poderoso, mapeando as entidades essenciais e suas relações.

| Nó (Node) | Descrição | Atributos Exemplo |
| :--- | :--- | :--- |
| `(:User)` | Usuários do sistema. | `name`, `state` |
| `(:Track)` | As músicas. | `name`, `year`, `energy`, `danceability` |
| `(:Artist)` | Os intérpretes das músicas. | `name`, `artistId` |
| `(:Genre)` | Os estilos musicais. | `name` |

| Relacionamento | De/Para | Propriedades Exemplo |
| :--- | :--- | :--- |
| `[:LISTENED_TO]` | `(:User) -> (:Track)` | `play_count` (contador de reproduções) |
| `[:LIKES]` | `(:User) -> (:Artist)` | (Preferência forte) |
| `[:PERFORMED_BY]` | `(:Track) -> (:Artist)` | |
| `[:HAS_GENRE]` | `(:Track) -> (:Genre)` | |

---

## 🛠️ Script de Inicialização (Criação de Dados Simulados)

Devido a problemas na importação de CSVs, o grafo foi populado manualmente com 30 Artistas, 30 Músicas, 5 Gêneros e interações iniciais simuladas para 3 usuários.

> **Instrução:** Execute os comandos abaixo no seu Neo4j Browser.

### 1. Criação de Gêneros, Artistas e Músicas

```cypher
// Criar Gêneros
MERGE (g1:Genre {name: 'Rock'});
MERGE (g2:Genre {name: 'Pop'});
MERGE (g3:Genre {name: 'Hip Hop'});
MERGE (g4:Genre {name: 'Eletronica'});
MERGE (g5:Genre {name: 'Jazz'});

// Criar Artistas (A-01 a A-30)
MERGE (a01:Artist {artistId: 'A-01', name: 'Queen'});
MERGE (a02:Artist {artistId: 'A-02', name: 'Adele'});
// ... (outros artistas)

// Criar Músicas (M-01 a M-30)
MERGE (m01:Track {trackId: 'M-01', name: 'Bohemian Rhapsody', year: 1975, energy: 0.8});
MERGE (m02:Track {trackId: 'M-02', name: 'In The End', year: 2000, energy: 0.9});
// ... (outras músicas com atributos)

```

### 2. Conexões: Gêneros e Artistas

```cypher
// Exemplo de Conexão Gênero (Música -> Gênero)
MATCH (m:Track {name: "Bohemian Rhapsody"}), (g:Genre {name: "Rock"})
MERGE (m)-[:HAS_GENRE]->(g);

// Exemplo de Conexão Artista (Música -> Artista)
MATCH (m:Track {trackId: 'M-01'}), (a:Artist {name: 'Queen'})
MERGE (m)-[:PERFORMED_BY]->(a);

// **NOTA:** As mais de 60 conexões de GÊNERO e ARTISTA foram omitidas aqui por brevidade.

```

### 3. Criação de Usuários e Interações


```cypher
// Criar Usuários
MERGE (u1:User {userId: 'u-alice', name: 'Alice', state: 'SP'});
MERGE (u2:User {userId: 'u-bob', name: 'Bob', state: 'RJ'});
MERGE (u3:User {userId: 'u-charlie', name: 'Charlie', state: 'MG'});

// Simulação de Escuta e Preferência da Alice
MATCH (u:User {name: 'Alice'}), (m:Track {trackId: 'M-01'})
MERGE (u)-[:LISTENED_TO {play_count: 5, timestamp: datetime()}]->(m);

MATCH (u:User {name: 'Alice'}), (a:Artist {name: 'Queen'})
MERGE (u)-[:LIKES]->(a);


```

## 🔎 Consultas de Recomendação (Cypher)
Abaixo estão 5 consultas estratégicas que demonstram o poder do grafo em diferentes tipos de recomendação:

**Pergunta 1: Recomendação Colaborativa (Amigos de Gosto)
O que recomendar a Alice, baseando-se no que outros usuários que gostam de Eletrônica mais escutaram?**


```cypher
MATCH (u_alice:User {name: 'Alice'}), (g:Genre {name: 'Eletronica'})
MATCH (u_other:User)-[l_other:LISTENED_TO]->(t_rec:Track)-[:HAS_GENRE]->(g) WHERE u_other <> u_alice
WHERE NOT EXISTS((u_alice)-[:LISTENED_TO]->(t_rec))
WITH t_rec.name AS RecommendedTrack, t_rec.artist AS Artist, SUM(l_other.play_count) AS TotalPlaysBySimilarUsers
RETURN RecommendedTrack, Artist, TotalPlaysBySimilarUsers ORDER BY TotalPlaysBySimilarUsers DESC LIMIT 5
```

Resultado: "Around the World" (Daft Punk) e "Scary Monsters and Nice Sprites" (Skrillex).

**Pergunta 2: Recomendação Baseada em Propriedades (Conteúdo)
Quais músicas com Danceability > 0.8 e lançadas após 2010 são ideais para festa?**


```cypher
MATCH (t:Track) WHERE t.danceability IS NOT NULL AND t.danceability > 0.8 AND t.year >= 2010
MATCH (t)-[:PERFORMED_BY]->(a:Artist)
MATCH (t)-[:HAS_GENRE]->(g:Genre)
RETURN t.name AS TrackName, a.name AS Artist, g.name AS Genre, t.danceability
ORDER BY t.danceability DESC, t.year DESC LIMIT 5

```

Resultado: "Levitating" da Artista "Dua Lipa".

**Pergunta 3: Influência e Popularidade Global
Qual artista possui o maior total de escutas (play_count) no sistema?**


```cypher
MATCH (a:Artist)<-[:PERFORMED_BY]-(t:Track)<-[l:LISTENED_TO]-(u:User)
WITH a.name AS ArtistName, SUM(l.play_count) AS TotalPlays
RETURN ArtistName, TotalPlays ORDER BY TotalPlays DESC LIMIT 5

```


Resultado: Eminem (10), seguido por Taylor Swift (8) e Louis Armstrong (7).

**Pergunta 4: Recomendação de Nicho (Exploração de Atributos)
Encontre as 5 músicas de Jazz mais acústicas (Acousticness > 0.95) lançadas antes de 1960.**


```cypher
MATCH (t:Track)-[:HAS_GENRE]->(g:Genre {name: 'Jazz'})
WHERE t.acousticness IS NOT NULL AND t.acousticness > 0.95 AND t.year < 1960
MATCH (t)-[:PERFORMED_BY]->(a:Artist)
RETURN t.name AS TrackName, a.name AS Artist, t.year AS ReleaseYear, t.acousticness
ORDER BY t.acousticness DESC LIMIT 5

```


Resultado: "So What" (Miles Davis), "Summertime" (Ella Fitzgerald), "Take The A Train" (Duke Ellington).

**Pergunta 5: Afinidade de Gênero (Gênero Favorito da Alice)
Quais são as 5 músicas mais recentes e com maior Energia do Gênero mais escutado pela Alice, que ela ainda não conhece?**


```cypher
MATCH (u:User {name: 'Alice'})-[l:LISTENED_TO]->(t:Track)-[:HAS_GENRE]->(g:Genre)
WITH u, g, SUM(l.play_count) AS TotalGenrePlays
ORDER BY TotalGenrePlays DESC LIMIT 1

MATCH (g)<-[:HAS_GENRE]-(t_rec:Track)
WHERE NOT (u)-[:LISTENED_TO]->(t_rec)

RETURN t_rec.name AS RecommendedTrack, t_rec.artist AS Artist, t_rec.year AS Year, t_rec.energy AS Energy
ORDER BY t_rec.energy DESC, t_rec.year DESC
LIMIT 5

```


Resultado: Músicas Pop com alta energia, como "Levitating", "Shape of You", "Diamonds" e "Halo".


## ⏭️ Próximos Passos e Evolução

Com essa base sólida em grafos, o caminho para um sistema de recomendação de nível de produção está aberto:

Integração de Dados: Conexão com bases de dados maiores via LOAD CSV ou ferramentas de integração.

Análise Avançada: Aplicação de Algoritmos GDS (Graph Data Science) como PageRank ou Similaridade de Jaccard para obter afinidades de usuário ainda mais precisas.

Este projeto demonstra como a modelagem em grafos simplifica e otimiza a lógica de recomendação complexa!
