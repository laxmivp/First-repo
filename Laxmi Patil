from Bio import Entrez
import csv

def fetch_and_filter_papers(query, output_filename="papers.csv"):
    """
    Fetches research papers from PubMed based on a query, filters for
    pharmaceutical/biotech affiliations, and saves the results to a CSV file.
    """

    Entrez.email = "your_email@example.com"  # Replace with your email

    # Search PubMed
    handle = Entrez.esearch(db="pubmed", term=query, retmax="200") # Increased retmax
    record = Entrez.read(handle)
    handle.close()
    pubmed_ids = record["IdList"]

    # Fetch paper details
    papers = []
    if pubmed_ids:
        handle = Entrez.efetch(db="pubmed", id=pubmed_ids, rettype="medline", retmode="text")
        records = handle.read()
        handle.close()

        # Process each paper
        for record in records.split("

"):
            if not record.strip():
                continue

            lines = record.splitlines()
            title = None
            authors = []
            abstract = None
            journal = None
            author_affiliations = {} # Store affiliations for each author

            # Parse the Medline record
            pmid = None
            for line in lines:
                if line.startswith("PMID- "):
                    pmid = line[6:].strip()
                elif line.startswith("TI  - "):
                    title = line[6:].strip()
                elif line.startswith("AU  - "):
                    authors.append(line[6:].strip())
                    author_affiliations[line[6:].strip()] = "" # Initialize affiliation
                elif line.startswith("AB  - "):
                    abstract = line[6:].strip()
                elif
