---
title: "Automating Cloud Operations & Infrastructure Tasks with Python"
summary: "How Python powers modern DevOps: from writing custom CLI tools with Typer and Click, to automating AWS cloud resource lifecycles with Boto3 and async workers."
categories: ["Python", "DevOps"]
tags: ["python", "automation", "boto3", "scripting", "cloud"]
date: 2026-05-02
draft: false
showTableOfContents: true
---

While Infrastructure as Code tools like Terraform excel at provisioning static resources, day-to-day operational tasks—such as rotating secrets, auditing orphan EBS volumes, auto-remediating security groups, and orchestrating deployment rollouts—require programmatic flexibility.

This is where **Python** shines as the ultimate DevOps language. Its rich ecosystem (`boto3`, `typer`, `pydantic`, `httpx`) allows engineers to write readable, testable, and maintainable automation scripts.

---

## 🛠️ 1. Building Clean DevOps CLI Utilities

Instead of brittle bash scripts with fragile regex, use **Typer** or **Click** to build intuitive command-line interfaces for internal engineering teams:

```python
import typer
from rich.console import Console
from rich.table import Table

app = typer.Typer(help="DevOps Cluster & Cloud Management CLI")
console = Console()

@app.command()
def audit_resources(
    region: str = typer.Option("us-east-1", "--region", "-r", help="AWS Region"),
    dry_run: bool = typer.Option(True, "--dry-run", help="Simulate execution without changes")
):
    """Scan and list unattached EBS volumes across accounts."""
    console.print(f"[bold cyan]Scanning region:[/bold cyan] {region} (Dry Run: {dry_run})")
    
    table = Table(title="Unattached EBS Volumes")
    table.add_column("Volume ID", style="cyan")
    table.add_column("Size (GB)", justify="right", style="magenta")
    table.add_column("Days Inactive", justify="right", style="green")

    # Example simulated findings
    table.add_row("vol-0a1b2c3d4e5f6g7h8", "100", "45")
    table.add_row("vol-9z8y7x6w5v4u3t2s1", "500", "90")

    console.print(table)

if __name__ == "__main__":
    app()
```

---

## ☁️ 2. Cloud Automation with Boto3

Here is a practical Python function utilizing `boto3` to audit and clean up untagged or stale cloud resources with strict error handling and safety checks:

```python
import boto3
from botocore.exceptions import ClientError
import logging

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger("cloud-janitor")

def cleanup_stale_snapshots(owner_id: str, days_threshold: int = 30):
    """Identify and safely purge unattached snapshots older than the threshold."""
    ec2 = boto3.client("ec2")
    try:
        paginator = ec2.get_paginator("describe_snapshots")
        page_iterator = paginator.paginate(OwnerIds=[owner_id])

        for page in page_iterator:
            for snapshot in page.get("Snapshots", []):
                snapshot_id = snapshot["SnapshotId"]
                logger.info(f"Evaluating snapshot: {snapshot_id}")
                # Business logic: evaluate retention and dependency tags
                
    except ClientError as err:
        logger.error(f"Failed to query EC2 snapshots: {err.response['Error']['Message']}")
        raise
```

---

## 🚀 3. Best Practices for DevOps Python Code

- **Type Annotations & Pydantic:** Always use type hints and Pydantic models for parsing API payloads and cloud responses to catch schema changes early.
- **Structured Logging:** Emit logs in JSON format so monitoring systems (DataDog, CloudWatch, Loki) can index them seamlessly.
- **Defensive Error Handling:** Handle transient network errors by configuring exponential backoff retries on all HTTP / Boto3 clients.
- **Unit Testing:** Write unit tests for your automation scripts using `pytest` and mock cloud APIs using `moto` or `responses`.

By treating automation scripts with the same engineering rigor as production applications, DevOps teams eliminate fragile manual tasks and ensure long-term stability.