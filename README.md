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
