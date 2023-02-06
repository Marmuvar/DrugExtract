# DrugExtract

## Overview

My [thesis studying pharmaceutical competition](https://github.com/Marmuvar/GenRxPredict/blob/main/Benmuvhar-Classification%20of%20Generic%20Manufacturers%20and%20Competition%20in%20the%20Pharmaceutical%20Industry.pdf) relied upon data compiled from annual reports from the United States Food and Drug Administration.  The DrugExtract module describes the data sources and custom python code used to parse .pdf format files.  From these, individual line items are extracted from whitespace-delimited tables having irregular entries. Subsequent steps eliminate duplicates and perform limited data cleaning.

## Background

Each year, the FDA publishes the commonly referenced "Orange Book" [@fda05], an official record of all drug products approved for marketing in the United States. This book establishes which products are generic and substitutable for each innovative brand product.  It contains a record of all products discontinued from marketing.  Finally, it details which patents the brand products claim for each drug product.  These specific patents may form the basis for any infringement lawsuits brought against generic applicants in accordance with the Patent Term Restoration Act.  Details of the patents also include any regulatory exclusivity clauses that accompany each patent.  

These sections differ in format and content.  Common details in each section include the drug substance name, unique application number, and product code, which enumerates differing product strengths or other variations.  Together, these establish a unique key for correlating distinct records across sections.  A sample page from the drug product listing appears here:

![OB_example](images/OB_example.png)

The Prescription Drug Product section includes the following fields:

| Field | Description |
| --- | --- |
| **Route of Administration:** | The FDA uses specific terms for how a product is introduced to the body.  This broadly differentiates products taken by mouth (oral), injected into the bloodstream (intravenous), and similar demarcations. |

| **Dosage Form:** | Products intended for each route of administration may take different forms.  For example, orally administered products comprises both liquids, tablets, and capsules taken by mouth. |

| **Drug Product Name:** |Commercial names for the product may either be an invented trade name for the brand product, such as the cholesotrol drug Lipitor, or the name of the drug substance used in the generic product, such as acetaminophen, found in the pain treatment Tylenol.  Because the product names are not unique, they can be ignored in the analysis for this report. |

| **Reference Designation:** | For each unique drug substance, route of administration, and dosage form, the FDA must designate one or more products as the reference product for subsequent generic products to test against.  In certain cases, a separate product receives designation as a reference standard due to discontinuance of the original reference product or the replacement of the reference product with newer products that follow modern application requirements. |

|**Therapeutic Equivalence Code:** |Where multiple product sources exist for a drug substance and route of administration, the FDA assigns a two letter equivalence code, such as "AB".  Generally, the first letter represents whether product substitutions are allowed ("A") or forbidden ("B"), while the second letter represents the route of administration.  Due to differences in physiological absorption or performance, a suffixed digit may differentiate product classes within the same drug substance and administration route category.  Similarly, differences in product performance may lead to the FDA adjudicating that a generic cannot be substituted for the brand. |

| **Sponsor Name:** | A product's sponsor is responsible for submission and maintenance of the application.  Due to changes in corporate identity due to merger or restructuring, the sponsor name may change across editions of the Orange Book. |

| **Approval Date:** | The approval date occurs when the FDA provides marketing authorization for a product after application review. |   

The discontinued product section duplicates entries of the prescription drug product list except that bioequivalence codes are unlisted.  Individual product codes and strengths under an application may be withdrawn from the market while others remain marketed.  The year of discontinuance is not reported and must be implied from the Orange Book edition in which the item first occurs. |   

The section entitled Prescription and OTC Drug Product Patent and Exclusivity List enumerates the following details:

| Field | Description |
| --- | --- |
| **Patent Number and Expiration Date:** | For each application and product number (strength), the assigned patent number and patent expiration date are listed.  In rare cases, patent numbers may not apply to each strength in an application.  Multiple patents may be assigned to a single drug product; |
| **Patent Codes:** |  Codes describe if a patent pertains to the drug substance, the drug product (dosage form), or therapeutic usage area.  The usage code may be one of over 3000 four digit codes that correspond to a brief description of the clinical or treatment aspects claimed by a patent.  For example: "U-2627: Topical treatment of plaque psoriasis in patients 12 years and older."  Multiple usage codes may be assigned to a single patent; |

| **Exclusivity Codes and Expiration Date:**  | Exclusivity periods represent statutory prohibitions that prevent the FDA from approving generic products for a certain time period.  These arise from laws enabling new molecule development, treatment of rare diseases, clinical studies in pediatric populations, and other reasons.  Exclusivity periods may run concurrently or in serial with patents.  Multiple exclusivity codes may be assigned to a single patent.  Certain exclusivity codes, such as those for pediatric exclusivity or orphan drug designations, may be assigned to the drug product rather than a patent and are listed as independent line items.  |

While the FDA maintains an electronic database of the Orange Book, the online version only captures data for the current year. Certain details are deprecated following periodic updates. As patents expire, they are removed from the online listing. As products are removed from the market, the equivalence relationship between branded and generic products is removed. To develop a comprehensive review of historical applications, it was necessary to perform data extraction from PDF renditions of the Orange Book. Historical editions were made available online via a regulatory law firm (Hyman, Phelps, McNamarra) following a Freedom of Information Act request to the FDA for electronic copies.  Due to illegibility of the original scans, data could not be extracted from editions prior to the 25th edition.  However, an archival project existed online that captured key elements of the patent history (Drug Database, N.D).  Researchers for this database had also encountered legibility challenges and had performed manual entry.  In turn, a portion of these entries were manually against the Orange Book and acceptable supplements for the extracted patent information.   

## Data Acquisition  
Data extraction and limited cleaning was first performed using custom Python applications. Separate applications were to parse product labels, active product records, discontinued product records, and patent information.  These applications implemented standard libraries including numpy, pandas, os, and re.  Other packages are discussed in the descriptions of each application.  

Compressed product label zip files were obtained from the National Library of Medicine's archive.  Applications to traverse the zip files and parse the product labels were readily constructed, as the XML format creates a heierarchical structure of name tags and information.  Both zipfile and xml.etree packages used for this are in the standard Python library.  Following the schema outlined in the FDA's guidance document, inactive ingredients and packaging components were extracted from the relevant label sections.  

In brief, the extraction algorithm iterated across .zip files in a directory.  Each .zip file was opened, and the contained .xml file was opened for processing.  Separately, the XML tree was navigated to obtain a dataframe containing product identifiers with either packaging components or a list of ingredients. During processing of each file, the results for packaging or ingredient components were maintained in a list of dataframes.  Some variance was observed in the schema for ingredients, resulting in a minority of cases using the tag "inactiveIngredientSubstance" rather than "ingredient".  The algorithm was modified to capture both cases.  Once all files were processed, the list of dataframes was concatenated into a single data frame, and the results were written to a csv file for downstream processing.  Delaying combination of dataframes until the last step improved program efficiency because of the lower computational overhead associated with working with the simpler list structure during each iteration [@p21].   
In contrast, the Orange Book files presented numerous challenges in format and content.  For each section, data groups reflected a heirarchical structure based on labeled headers.  Considering the Pharmaceutical Product section, each group of drug substance, route of administration, and drug product headers were followed by line item details for relevant applications grouped by sponsor.  Columns were maintained using consistent white space between entries.  For each different strength, sponsor and header information were not repeated.  Further complicating this, sponsor names and product strengths could require multiple lines, resulting in extraneous white space.  Last, the spacing of columns between Orange Book editions varied.   Because the contents and size of each grouping varied, each page was effectively unique.  Therefore, data extraction required tables to be parsed row-wise to collect required cells.  

A custom application using tabula [@f19] and [@p22]] was designed.  tabula provides a flexible tool for extracting PDF data from a defined table or field format.  Although the variable field size prevented full use of the package, it provided a valuable tool for converting column and row based data into a dataframe.  Similarly, pypdf2 provides basic tools for identifying and extracting pages.  While a description of the process is provided here for extracting drug product details, the overall algorithm is applicable with modification for fields of the discontinued product and patents sections.   

Column spacing for each .pdf file was manually determined and used as a predefined value for the algorithm.  For each pdf file, the algorithm first identified the start and ending page for the Pharmaceutical Product section based on page header information.  Within this range, individual rows containing data fields were extracted.  In a second step, the extracted details were parsed.  A temporary line item was created to store heading elements (drug substance, route of administration, and drug product name).  Subsequent lines were parsed to collect remaining fields.  If the following line was a partial entry due to long sponsor or strength entries, the contents were appended to relevant fields.  If not, the entry was appended to the temporary line, and the temporary line was added to the master data frame.  If the following line contained new header details, the temporary line was further updated with new product details.  

## Data Cleaning and Preprocessing  

The ingredient and packaging information were usable as described in the data acquisition section.  Additional steps for cleaning, consistency, and consolidation were performed on data extracted from the Orange Book.  Cleaning steps ensured similar punctuation used for separators and eliminated excess whitespace from fields.  Also, legal entity designations "CORP", "CO", "INC", "LTD", "LLC", and "GLOBAL" were removed from sponsor names.  While these terms may reflect changes in underlying operating principles or business units in a company, it was decided to minimize identity changes related to a legal status.  Consistency steps addressed FDA's gradual standardization of dosage form descriptions, administrative updates, and changes in company names.  Due to a wide range of administrative routes listed for injectable, intravenous, and oral products, a number of low frequency categories were reduced to more general classifications.  Similar condensation of oral dosage form routes were made.  Changes in entries occurring across editions are described in Table \@ref(tab:Constraints).  Last, consolidation steps eliminated duplicate entries present following the previous standardization steps.  

<!--ts-->

| Constraint | Rationale |  
| --- | --- |
| Distinct products are defined by the application number and route of administration. | Establishes a consistent drug product identification |
| Duplicated entries based on application number, product number, and drug substance are limited to the first occurrence | Eliminates redundancy of the data file while recognizing status quo at time of application submission. |
| Where multiple strengths are issued to an application on multiple dates, only the earliest date will be considered|The thesis considers only the primary activities required for the initial drug approval. Adding a drug strength to an application often relies on significant existing research. Further, they are less likely to be impacted by patents due to elapsed time for patent expiry. |
| Where the sponsor’s name changes due to a company action, the original name of the applicant will be retained. | The thesis considers only the corporate identity at the time of approval. Although the industry has trended towards consolidation, the ability of smaller companies to produce generic products contrasts efficiencies established at larger companies. |
| For predictive models, only products for which the original branded product remains marketed will be considered. | This ensures formulation information is available based on the removal of discontinued products from the label database. |
| For predictive models, where multiple reference products are available, the product with the earliest approval date will be used as a reference. | This ensures a consistent starting point to determine opportunities for generic entry. |

<!--te-->

