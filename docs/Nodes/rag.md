# 🪿 Rag node

## Introduction
The RAG Node is a vital component in the language chain framework, specializing in parsing HTML content from documents. It efficiently extracts relevant information using user-defined tags, enhancing the processing of large HTML documents. By integrating parsing techniques like BeautifulSoupTransformer, the RAG Node optimizes document processing pipelines, contributing to improved efficiency and effectiveness in text processing tasks.


The implementation of the class is in this [link](https://github.com/VinciGit00/Scrapegraph-ai/blob/main/scrapegraphai/nodes/rag_node.py)
## Usage
```python
from scrapegraphai.models import Ollama
from scrapegraphai.nodes import RAGNode

graph_config = {
    "llm": {
        "model": "ollama/mistral",
        "temperature": 0,
        "format": "json",  # Ollama needs the format to be specified explicitly
        # "model_tokens": 2000, # set context length arbitrarily
        "base_url": "http://localhost:11434",
    },
    "embeddings": {
        "model": "ollama/nomic-embed-text",
        "temperature": 0,
        "base_url": "http://localhost:11434",
    }
}

llm_model = Ollama(graph_config["llm"])

rag_node = RAGNode(
    input="user_prompt & (parsed_doc | doc)",
    output=["relevant_chunks"],
    node_config={"llm": llm_model},
)
```
## Implementation
```python
"""
Module for parsing the HTML node
"""

from typing import List
from langchain.docstore.document import Document
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import EmbeddingsFilter, DocumentCompressorPipeline
from langchain_community.document_transformers import EmbeddingsRedundantFilter
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings
from .base_node import BaseNode


class RAGNode(BaseNode):
    """
    A node responsible for compressing the input tokens and storing the document
    in a vector database for retrieval.

    It allows scraping of big documents without exceeding the token limit of the language model.

    Attributes:
        node_name (str): The unique identifier name for the node, defaulting to "ParseHTMLNode".
        node_type (str): The type of the node, set to "node" indicating a standard operational node.

    Args:
        node_name (str, optional): The unique identifier name for the node. 
        Defaults to "ParseHTMLNode".

    Methods:
        execute(state): Parses the HTML document contained within the state using 
        the specified tags, if provided, and updates the state with the parsed content.
    """

    def __init__(self, input: str, output: List[str], model_config: dict, node_name: str = "RAG"):
        """
        Initializes the ParseHTMLNode with a node name.
        """
        super().__init__(node_name, "node", input, output, 2, model_config)
        self.llm_model = model_config["llm_model"]

    def execute(self, state):
        """
        Executes the node's logic to implement RAG (Retrieval-Augmented Generation) 
        The method updates the state with relevant chunks of the document.

        Args:
            state (dict): The state containing the 'document' key with the HTML content

        Returns:
            dict: The updated state containing the 'relevant_chunks' key with the relevant chunks.

        Raises:
            KeyError: If 'document' is not found in the state, indicating that the necessary 
                      information for parsing is missing.
        """

        print(f"--- Executing {self.node_name} Node ---")

        # Interpret input keys based on the provided input expression
        input_keys = self.get_input_keys(state)

        # Fetching data from the state based on the input keys
        input_data = [state[key] for key in input_keys]

        user_prompt = input_data[0]
        doc = input_data[1]

        chunked_docs = []

        for i, chunk in enumerate(doc):
            doc = Document(
                page_content=chunk,
                metadata={
                    "chunk": i + 1,
                },
            )
            chunked_docs.append(doc)

        print("--- (updated chunks metadata) ---")

        openai_key = self.llm_model.openai_api_key
        retriever = FAISS.from_documents(chunked_docs,
                                         OpenAIEmbeddings(api_key=openai_key)).as_retriever()
        # could be any embedding of your choice
        embeddings = OpenAIEmbeddings(api_key=openai_key)
        redundant_filter = EmbeddingsRedundantFilter(embeddings=embeddings)
        # similarity_threshold could be set, now k=20
        relevant_filter = EmbeddingsFilter(embeddings=embeddings)
        pipeline_compressor = DocumentCompressorPipeline(
            transformers=[redundant_filter, relevant_filter]
        )
        # redundant + relevant filter compressor
        compression_retriever = ContextualCompressionRetriever(
            base_compressor=pipeline_compressor, base_retriever=retriever
        )

        # relevant filter compressor only
        # compression_retriever = ContextualCompressionRetriever(
        #     base_compressor=relevant_filter, base_retriever=retriever
        # )

        compressed_docs = compression_retriever.get_relevant_documents(
            user_prompt)

        print("--- (tokens compressed and vector stored) ---")

        state.update({self.output[0]: compressed_docs})
        return state
```