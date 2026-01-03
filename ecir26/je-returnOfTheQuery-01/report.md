# Retrieval System - Return of the Query

## Reranking

In der parallel laufenden Vorlesung wurde **Reranking** als erste Verbesserung eines bestehenden Systems vorgeschlagen. 

Beim Reranking wird das Ergebnis eines bestehenden Retrieval-Systems mit einem anderen, teureren System verglichen, um es zu verbessern. Der Begriff „teuer” bezieht sich hierbei auf die Computing-Kosten, die das System verursacht. In der Regel ist durch die höheren Kosten ein besseres Ergebnis zu erwarten. Um den Aufwand jedoch in einem gewissen Rahmen zu halten, wird nicht das gesamte Ergebnis neu bewertet, sondern nur ein Teil davon. Typischerweise wird auf den ersten 100 Ergebnissen rerankt und, falls erneut rerankt wird, auf den ersten 5 bis 50.

### Learning to Rank (LTR)

Es gibt verschiedene Ansätze, um das Retrieval zu erlernen. Dabei gibt es drei Ziele.

- **Pointwise LTR**: Es werden die Scores der Dokumente für jedes Dokument individuell vorhergesagt. Bei der Pointwise LTR wird [monoT5 [Nogueira et al., 2020]](https://arxiv.org/pdf/2003.06713) genutzt. 

- **Pairwise LTR**: Es werden "präfferenzen" für Paare von Doumenten vorhergesagt. Es wird [duoT5 [Pradeep et al., 2021]](https://arxiv.org/pdf/2101.05667) genutzt.

- **Listwise LTR**: Es wird ein Rank effektivitäts Measure optimiert. Dies ist nach Stand der vorlesung noch nicht implementiert.

Ein typischer Ansatz für eine Reranking-Pipeline ist der folgende:
1. BM25-Ranking auf der kompletten Sammlung,
2. Pointwise Reranking der Top-100-Ergebnisse.
3. Pairwise Reranking der Top 50 Ergebnisse aus Schritt 2.
4. Ranking nach dem aggregierten Score aus Schritt 3.

Dieser Ansatz wird auch in unserem System mit anderen Top-Score-Ergebnissen verwendet.

## Code

Um das Rad nicht neu zu erfinden, nutzen wir die Python-Retrieval-Bibliothek [Pyterrier](https://github.com/terrier-org/pyterrier.git). Pyterrier bietet viele bereits implementierte und optimierte Retrieval-Methoden und -Ansätze. Standardmäßig ist BM25 enthalten.

Falls zur Laufzeit noch kein Index existiert, wird dieser im Zuge der `get_index()` Funktion erstellt. Dabei können verschiedene stemmer verwendet werden.

```python
def get_index(dataset, field, output_path):
    index_dir = output_path / "indexes" / f"{dataset}-on-{field}"
    # ...

    with tracking(export_file_path=index_dir / "index-metadata.yml", export_format=ExportFormat.IR_METADATA):
        # Verwendung der in Pyterrier verfügbaren Implementierung des Porter-Stemming Algorithmus
        pt.IterDictIndexer(str(index_dir.absolute()), meta={'docno' : 100}, verbose=True, stemmer="PorterStemmer").index(docs)

    return pt.IndexFactory.of(str(index_dir.absolute()))
```

Pointwise-Modelle sind verfügbar, Pairwise-Modelle jedoch nicht. Dafür gibt es die Bibliothek [Pyterrier T5](https://github.com/terrierteam/pyterrier_t5?tab=readme-overview). Diese implementiert die monoT5- und die duoT5-Modelle sowie die Pointwise- und die Pairwise-Modelle. Diese können direkt genutzt werden.

```python
import pyterrier
import pyterrier_t5

# Initialisierung der Retrieval Modelle
# Für die einfachheit wird der Index nicht definiert.
bm25 = pt.terrier.Retriever(index, wmodel="BM25")
reranker_pointwise = MonoT5ReRanker()
reranker_pairwise = DuoT5ReRanker()
```

In Pyterrier gibt es verschiedene Operatoren, mit denen sich Aussagen vereinfachen lassen. Ein Beispiel ist der Operator [>>](https://pyterrier.readthedocs.io/en/latest/operators.html#then). Dieser ermöglicht die Erstellung von Pipelines und somit die übersichtliche Definition mehrerer Arbeitsschritte hintereinander in einer Zeile.

```python
# Rerank Pipeline
mono_pipeline = retriever % 100 >> reranker_pointwise
duo_pipeline = mono_pipeline % 5 >> reranker_pairwise
run = duo_pipeline.transform(topics)
```

## Ergebnis

### Ein Erster Ansatz

Zu Beginn haben wir die Baseline des Systems mit einem BM25-Retrieval-System gemessen. Als Gütemaß haben wir NDCG@10 gewählt. Dieser Wert wurde für alle berechnet und anhand dessen wurden die Vergleiche durchgeführt.

| |Modell|  ndcg@10|
|-|-|-|
|-|                           Baseline BM25| 0.451635|
|0|        pyterrier-DirichletLM-on-title-3| 0.110035|
|1|         pyterrier-BM25-on-description-3| 0.162492|
|2|  pyterrier-DirichletLM-on-description-3| 0.065785|
|3| pyterrier-DirichletLM-on-default_text-3| 0.247526|
|4|        pyterrier-BM25-on-default_text-3| 0.317093|
|5|               pyterrier-BM25-on-title-3| 0.274660|
|6|         pyterrier-PL2-on-default_text-3| 0.346148|
|7|          pyterrier-PL2-on-description-3| 0.139911|
|8|                pyterrier-PL2-on-title-3| 0.263840|

Wie in der Tabelle zu sehen ist, wurden die Baseline-Werte unterschritten. Das System hat sich durch das Reranking nicht verbessert, sondern verschlechtert. Die Gründe dafür können wir uns derzeit nicht erklären.

### Verwendung von Stemming und optimierung des Rerankers

Durch experimentelle Versuche mit den Werten der mono / duo pipeline konnten die Ergebnisse des Retrieval-Systems (teilweise) verbessert werden. Während 

```python
# Rerank Pipeline
mono_pipeline = retriever % 100 >> reranker_pointwise
duo_pipeline = mono_pipeline % 10 >> reranker_pairwise
run = duo_pipeline.transform(topics)
```

<table>
<tr><th><div align="center">mono 100 / duo 5</th><th><center>mono 100 / duo 10</th></tr>
<tr><td>

| |Modell|  ndcg@10|
|-|-|-|
|0|        pyterrier-DirichletLM-on-title-3| 0.110035|
|1|         pyterrier-BM25-on-description-3| 0.162492|
|2|  pyterrier-DirichletLM-on-description-3| 0.065785|
|3| pyterrier-DirichletLM-on-default_text-3| 0.247526|
|4|        pyterrier-BM25-on-default_text-3| 0.317093|
|5|               pyterrier-BM25-on-title-3| 0.274660|
|6|         pyterrier-PL2-on-default_text-3| 0.346148|
|7|          pyterrier-PL2-on-description-3| 0.139911|
|8|                pyterrier-PL2-on-title-3| 0.263840|

</td><td>

| |Modell|  ndcg@10|
|-|-|-|
|0|        pyterrier-DirichletLM-on-title-3| 0.152959|
|1|         pyterrier-BM25-on-description-3| 0.180296|
|2|  pyterrier-DirichletLM-on-description-3| 0.110791|
|3| pyterrier-DirichletLM-on-default_text-3| 0.282637|
|4|        pyterrier-BM25-on-default_text-3| 0.316709|
|5|               pyterrier-BM25-on-title-3| 0.302422|
|6|         pyterrier-PL2-on-default_text-3| 0.414098|
|7|          pyterrier-PL2-on-description-3| 0.172011|
|8|                pyterrier-PL2-on-title-3| 0.279087|

</td></tr> </table>

Zusätzlich dazu wurden beim Bau des Index verschiedene Stemmer ausprobiert. Die besten Ergebnisse lieferte der in Pyterrier implementierte Porter-Stemming-Algorithmus, wobei die einzige Veränderung bei BM25 mit default text auftritt.

<table>
<tr><th><div align="center">Kein Stemming</th><th><center>Porter Stemming</th></tr>
<tr><td>

| |Modell|  ndcg@10|
|-|-|-|
|0|        pyterrier-DirichletLM-on-title-3| 0.152959|
|1|         pyterrier-BM25-on-description-3| 0.180296|
|2|  pyterrier-DirichletLM-on-description-3| 0.110791|
|3| pyterrier-DirichletLM-on-default_text-3| 0.282637|
|4|        pyterrier-BM25-on-default_text-3| 0.316709|
|5|               pyterrier-BM25-on-title-3| 0.302422|
|6|         pyterrier-PL2-on-default_text-3| 0.414098|
|7|          pyterrier-PL2-on-description-3| 0.172011|
|8|                pyterrier-PL2-on-title-3| 0.279087|

</td><td>

| |Modell|  ndcg@10|
|-|-|-|
|0|        pyterrier-DirichletLM-on-title-3| 0.152959|
|1|         pyterrier-BM25-on-description-3| 0.180296|
|2|  pyterrier-DirichletLM-on-description-3| 0.110791|
|3| pyterrier-DirichletLM-on-default_text-3| 0.282637|
|4|        pyterrier-BM25-on-default_text-3| 0.370291|
|5|               pyterrier-BM25-on-title-3| 0.302422|
|6|         pyterrier-PL2-on-default_text-3| 0.414098|
|7|          pyterrier-PL2-on-description-3| 0.172011|
|8|                pyterrier-PL2-on-title-3| 0.279087|

</td></tr> </table>

## Contributers

Team: Return of the Query
- [Moritz Raetz](mailto:moritz.raetz@uni.jena.de)
- [Leonard Teschner](mailto:leonard.teschner@uni-jena.de)

Text mit [Deepl.com/write](https://www.deepl.com/de/write) umgeschrieben.