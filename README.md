import requests
import csv
import time
import re
import os
from dotenv import load_dotenv

# Load environment variables (for API keys)
load_dotenv()

class ResearchPaperFetcher:
    def __init__(self):
        # API keys should be stored in environment variables for security
        self.semantic_scholar_api_key = os.getenv("SEMANTIC_SCHOLAR_API_KEY")
        self.headers = {
            "x-api-key": self.semantic_scholar_api_key
        }
        
        # List of common pharmaceutical and biotech company keywords
        # This list can be expanded or replaced with a more comprehensive database
        self.pharma_biotech_keywords = [
            "pfizer", "merck", "novartis", "roche", "sanofi", "johnson & johnson", 
            "glaxosmithkline", "gsk", "astrazeneca", "abbvie", "amgen", "gilead", 
            "biogen", "regeneron", "moderna", "biontech", "vertex", "alexion",
            "pharma", "biotech", "therapeutics", "biosciences", "pharmaceuticals"
        ]
    
    def search_papers(self, query, limit=100):
        """
        Search for papers using the Semantic Scholar API
        """
        base_url = "https://api.semanticscholar.org/graph/v1/paper/search"
        params = {
            "query": query,
            "limit": limit,
            "fields": "title,authors,year,venue,url,abstract"
        }
        
        try:
            response = requests.get(base_url, params=params, headers=self.headers)
            response.raise_for_status()
            return response.json().get('data', [])
        except requests.exceptions.RequestException as e:
            print(f"Error fetching papers: {e}")
            return []
    
    def get_author_details(self, author_id):
        """
        Get author details including affiliations
        """
        url = f"https://api.semanticscholar.org/graph/v1/author/{author_id}"
        params = {
            "fields": "name,affiliations,homepage"
        }
        
        try:
            response = requests.get(url, params=params, headers=self.headers)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.RequestException as e:
            print(f"Error fetching author details: {e}")
            return {}
    
    def is_pharma_biotech_affiliation(self, affiliation):
        """
        Check if an affiliation is from a pharmaceutical or biotech company
        """
        if not affiliation:
            return False
            
        affiliation_lower = affiliation.lower()
        
        # Check if any of the pharma/biotech keywords are in the affiliation
        for keyword in self.pharma_biotech_keywords:
            if keyword in affiliation_lower:
                return True
                
        return False
    
    def filter_papers_by_author_affiliation(self, papers):
        """
        Filter papers to include only those with at least one author from a pharma/biotech company
        """
        filtered_papers = []
        
        for paper in papers:
            has_pharma_author = False
            pharma_authors = []
            
            for author in paper.get('authors', []):
                # Skip if author has no ID
                if not author.get('authorId'):
                    continue
                    
                # Get detailed author information including affiliations
                author_details = self.get_author_details(author.get('authorId'))
                time.sleep(1)  # Respect API rate limits
                
                affiliations = author_details.get('affiliations', [])
                
                # Check each affiliation
                for affiliation in affiliations:
                    if self.is_pharma_biotech_affiliation(affiliation):
                        has_pharma_author = True
                        pharma_authors.append({
                            'name': author.get('name'),
                            'affiliation': affiliation
                        })
                        break
            
            if has_pharma_author:
                paper_data = {
                    'title': paper.get('title', ''),
                    'year': paper.get('year', ''),
                    'venue': paper.get('venue', ''),
                    'url': paper.get('url', ''),
                    'abstract': paper.get('abstract', ''),
                    'pharma_authors': pharma_authors
                }
                filtered_papers.append(paper_data)
                
        return filtered_papers
    
    def export_to_csv(self, papers, filename="pharma_research_papers.csv"):
        """
        Export filtered papers to a CSV file
        """
        if not papers:
            print("No papers to export.")
            return
            
        try:
            with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
                fieldnames = ['title', 'year', 'venue', 'url', 'abstract', 'pharma_authors']
                writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
                
                writer.writeheader()
                for paper in papers:
                    # Convert pharma_authors list to a string representation
                    paper['pharma_authors'] = '; '.join([f"{author['name']} ({author['affiliation']})" 
                                                        for author in paper['pharma_authors']])
                    writer.writerow(paper)
                    
            print(f"Successfully exported {len(papers)} papers to {filename}")
        except Exception as e:
            print(f"Error exporting to CSV: {e}")
    
    def run(self, query):
        """
        Main method to execute the paper fetching process
        """
        print(f"Searching for papers matching query: '{query}'")
        papers = self.search_papers(query)
        print(f"Found {len(papers)} papers. Filtering by author affiliation...")
        
        filtered_papers = self.filter_papers_by_author_affiliation(papers)
        print(f"Filtered to {len(filtered_papers)} papers with pharmaceutical/biotech authors")
        
        filename = f"{query.replace(' ', '_')}_pharma_papers.csv"
        self.export_to_csv(filtered_papers, filename)
        
        return filtered_papers

if __name__ == "__main__":
    # Get user query
    user_query = input("Enter your research paper search query: ")
    
    # Initialize and run the fetcher
    fetcher = ResearchPaperFetcher()
    fetcher.run(user_query)
import requests
import csv
import argparse
import sys
import xml.etree.ElementTree as ET

# Function to fetch research papers from PubMed API
def fetch_papers(query):
    base_url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
    params = {
        'db': 'pubmed',
        'term': query,
        'retmode': 'xml',
        'retmax': 100  # Number of results to fetch (you may adjust as needed)
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code != 200:
        print(f"Error fetching papers: {response.status_code}", file=sys.stderr)
        return []
    
    # Parse XML response to extract PubMed IDs
    tree = ET.ElementTree(ET.fromstring(response.content))
    root = tree.getroot()
    pm_ids = [id_elem.text for id_elem in root.findall('.//Id')]
    
    return pm_ids

# Function to fetch details for each paper
def fetch_details(pm_id):
    base_url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"
    params = {
        'db': 'pubmed',
        'id': pm_id,
        'retmode': 'xml'
    }
    
    response = requests.get(base_url, params=params)
    if response.status_code != 200:
        print(f"Error fetching details for {pm_id}: {response.status_code}")
        return None
    
    # Parse XML response to extract required details
    tree = ET.ElementTree(ET.fromstring(response.content))
    root = tree.getroot()
    
    title = root.findtext('.//ArticleTitle', default='N/A')
    pub_date = root.findtext('.//PubDate', default='N/A')
    authors = root.findall('.//Author')
    non_academic_authors = []
    company_affiliations = []
    corresponding_email = 'N/A'
    
    for author in authors:
        affiliation = author.findtext('.//Affiliation', default='N/A')
        if 'pharmaceutical' in affiliation.lower() or 'biotech' in affiliation.lower():
            non_academic_authors.append(author.findtext('.//LastName') + ', ' + author.findtext('.//ForeName'))
            if 'pharmaceutical' in affiliation.lower():
                company_affiliations.append(affiliation)
        if author.findtext('.//Correspondence') == 'Y':
            corresponding_email = author.findtext('.//Email', default='N/A')
    
    return {
        'PubmedID': pm_id,
        'Title': title,
        'Publication Date': pub_date,
        'Non-academic Author(s)': '; '.join(non_academic_authors),
        'Company Affiliation(s)': '; '.join(company_affiliations),
        'Corresponding Author Email': corresponding_email
    }

# Main function to parse arguments, perform fetching and write CSV
def main():
    parser = argparse.ArgumentParser(description="Fetch research papers from PubMed API.")
    parser.add_argument('query', type=str, help='Search query for fetching papers.')
    parser.add_argument('-d', '--debug', action='store_true', help='Print debug information.')
    parser.add_argument('-o', '--output', type=str, default='output.csv', help='Output CSV file name.')
    
    args = parser.parse_args()

    if args.debug:
        print("Debug mode enabled.")
        print(f"User query: {args.query}")

    pm_ids = fetch_papers(args.query)
    if not pm_ids:
        print("No papers found.")
        return

    papers_details = []
    for pm_id in pm_ids:
        if args.debug:
            print(f"Fetching details for PubMed ID: {pm_id}")
        details = fetch_details(pm_id)
        if details:
            papers_details.append(details)

    # Writing to CSV
    with open(args.output, mode='w', newline='', encoding='utf-8') as csvfile:
        fieldnames = ['PubmedID', 'Title', 'Publication Date', 'Non-academic Author(s)', 'Company Affiliation(s)', 'Corresponding Author Email']
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(papers_details)

    print(f"Results written to {args.output}")

if __name__ == "__main__":
    main()
# First-repo
This is my first repository created for testing purpose.
