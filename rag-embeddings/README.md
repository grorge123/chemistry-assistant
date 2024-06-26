# Create a vector store for chemistry knowledge

LLMs often lack domain knowledge to accurately answer highly specific questions. A common technique to supplement domain knowledge to LLMs is through the RAG technique. It works like the following.

1. The domain knowledge is segmented into chunks of text.
2. Each text segment is turned into a numeric vector, aka an embedding, by the LLM. The embedding represents the LLM's "understanding" of the text.
3. The embeddings are stored in a vector database.
4. When the user asks a question, we use the LLM to turn the question into an embedding too.
5. We will search the vector database for embeddings that are similiar to the user's question.
6. The original text of the search result embeddings will be added to the prompt for the LLM to answer the question.

This approach provides the LLM with the knowledge context and even source so that it can answer the question more accurately.

## Start the Qdrant vector database

I choose Qdrant as the vector database to store and manage the knowledge embeddings. With Docker, you can easily run Qdrant locally on a personal computer.

```
mkdir qdrant_storage
mkdir qdrant_snapshots

nohup sudo docker run -d -p 6333:6333 -p 6334:6334 \
    -v $(pwd)/qdrant_storage:/qdrant/storage:z \
    -v $(pwd)/qdrant_snapshots:/qdrant/snapshots:z \
    qdrant/qdrant
```

Add a vector collection called `chemistry_book`.

```
curl -X PUT 'http://localhost:6333/collections/chemistry_book' \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "vectors": {
      "size": 384,
      "distance": "Cosine",
      "on_disk": true
    }
  }'
```

> We are using 384 dimensions for the embedding model `all-MiniLM-L6-v2`.

Confirm that the collection is successfully created.

```
curl 'http://localhost:6333/collections/chemistry_book'
```

Note: you can delete a collection using the following request.

```
curl -X DELETE 'http://localhost:6333/collections/chemistry_book'
```

Note: you can create a snapshot of the collection as follows. The snapshot is a file in the `./qdrant_snapshots` directory, which can be shared across the Internet.

```
curl -X POST 'http://localhost:6333/collections/chemistry_book/snapshots'
```

## Install WasmEdge with GGML plugin

To generate the embeddings, I need to process the text using the fine-tuned LLM. WasmEdge is a extremely lightweight and cross-platform runtime for LLMs. I will use it here.

```
curl -sSf https://raw.githubusercontent.com/WasmEdge/WasmEdge/master/utils/install.sh | bash -s -- --plugins wasi_nn-ggml
source /home/azureuser/.bashrc
```

## Build the program to generate embeddings

The application to generate and store embeddings is modified from LlamaEdge examples. It reads a chemistry-related text file, and runs the LLM to generate an embedding for each paragraph of text. The embeddings and associated original text is then saved to Qdrant.

First, make sure that you have the Rust toolchain installed.

```
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32-wasi
```

You also need the [clang compiler tools](https://clang.llvm.org/get_started.html) installed for TLS / SSL support.

```
# This is for Ubuntu Linux
sudo apt install clang
```

Next compile the application in this directory to Wasm bytecode.

```
RUSTFLAGS="--cfg wasmedge --cfg tokio_unstable" cargo build --target wasm32-wasi --release
```

## Download an embedding model

```
curl -LO https://huggingface.co/second-state/All-MiniLM-L6-v2-Embedding-GGUF/resolve/main/all-MiniLM-L6-v2-ggml-model-f16.gguf
```

## Generate embeddings

Now, we can run the Wasm app to generate embeddings from a text file [chemistry.txt](chemistry.txt) and save to the Qdrant `chemistry_book` collection.

```
cp target/wasm32-wasi/release/create_embeddings.wasm .
wasmedge --dir .:. \
  --nn-preload embedding:GGML:AUTO:all-MiniLM-L6-v2-ggml-model-f16.gguf \
  create_embeddings.wasm embedding chemistry_book 384 chemistry.txt
```

After it is done, you can check the vectors and their associated payloads by their IDs. THe example below returns the first and last vectors in the `chemistry_book` collection.

```
curl 'http://localhost:6333/collections/chemistry_book/points' \
  -H 'Content-Type: application/json' \
  --data-raw '{
    "ids": [0, 1231],
    "with_payload": true,
    "with_vector": false
  }'
```

You can also pass the following options to the program.

* Using `-m` or `--maximum_context_length` to specify a context length in the CLI argument. That is to truncate and warn for each text segment that goes above the context length.
* Using `-s` or `--start_vector_id` to specify the start vector ID in the CLI argument. This will allow us to run this app multiple times on multiple documents on the same vector collection.

```
wasmedge --dir .:. \
  --nn-preload embedding:GGML:AUTO:all-MiniLM-L6-v2-ggml-model-f16.gguf \
   create_embeddings.wasm embedding chemistry_book 384 chemistry.txt -s 5 -m 1024
```

## Next step

Once you have the the vector collection with domain-specific knowledge, you can skip ahead to Step 3 of this tutorial to start an API server. But I recommend you to go through Step 2 first to create a finetuned LLM that is optimized for chemistry questions!



