.. _quickstart:

Quick Start
===========

This guide demonstrates how to fine-tune a Gemma2 2B model using LoRA with a toy dataset.

The script can also be found on `Github <https://github.com/AI-Hypercomputer/kithara/blob/main/examples/singlehost/quick_start.py>`_.

Prefer running in Colab? Check out the `this Colab Notebook instead <https://colab.sandbox.google.com/github/AI-Hypercomputer/kithara/blob/main/examples/colab/SFT_with_LoRA_Gemma2-2b.ipynb>`_.

Setup
-----
Log into HuggingFace and import required packages::

    from huggingface_hub import login
    login(token="your_hf_token", add_to_git_credential=False)

    import os
    os.environ["KERAS_BACKEND"] = "jax"
    import keras
    import ray
    from kithara import (
        KerasHubModel,
        Dataloader,
        Trainer,
        SFTDataset,
    )

.. tip::
    New to HuggingFace? First create an access token, `apply access <https://huggingface.co/google/gemma-2-2b>`_ to the Gemma2 HuggingFace model which will be used in this example.

Quick Usage
----------

1. Create the Model::

    model = KerasHubModel.from_preset(
        "hf://google/gemma-2-2b",
        precision="mixed_bfloat16",
        lora_rank=4,
    )
    
2. Prepare Dataset::

    dataset_items = [
        {
            "prompt": "What is your name?",
            "answer": "My name is Kithara",
        }
        for _ in range(1000)
    ]
    dataset = ray.data.from_items(dataset_items)
    train_ds, eval_ds = dataset.train_test_split(test_size=500)

3. Create Dataset and Optimizer::

    train_dataset = SFTDataset(
        train_ds,
        tokenizer_handle="hf://google/gemma-2-2b",
        max_seq_len=4096,
    )
    
    eval_dataset = SFTDataset(
        eval_ds,
        tokenizer_handle="hf://google/gemma-2-2b",
        max_seq_len=4096,
    )
    
    optimizer = keras.optimizers.AdamW(
        learning_rate=2e-4,
        weight_decay=0.01
    )

4. Create Dataloaders::

    train_dataloader = Dataloader(
        train_dataset,
        per_device_batch_size=1
    )
    
    eval_dataloader = Dataloader(
        eval_dataset,
        per_device_batch_size=1
    )

5. Initialize and Run Trainer::

    trainer = Trainer(
        model=model,
        optimizer=optimizer,
        train_dataloader=train_dataloader,
        eval_dataloader=eval_dataloader,
        steps=200, # You can also use epochs instead of steps
        eval_steps_interval=10,
        max_eval_samples=50,
        log_steps_interval=10,
    )
    
    trainer.train()

6. Test the Model::

    pred = model.generate(
        "What is your name?",
        max_length=30,
        tokenizer_handle="hf://google/gemma-2-2b",
        return_decoded=True
    )
    print("Tuned model generates:", pred)

Running This Example on Single Host
------------------------------------------------

Simple copy paste this script from the Github repo, and run it on your TPU VM::

    python examples/singlehost/quick_start.py


Running This Example on Multi-host
---------------------------------

Kithara works with any accelerator orchestrator. However, if you are new to distributed training, we provide guide for :doc:`multihost training with Ray <scaling_with_ray>`.

Once you set up a Ray cluster, clone the Github Repo, and run this example with your Ray Cluster::

    python ray/submit_job.py "python3.11 examples/multihost/ray/TPU/quick_start.py" --hf-token your_token


Next Steps
-----------

Check out the :doc:`Finetuning Guide <finetuning_guide>` to craft out your own finetuning job.
