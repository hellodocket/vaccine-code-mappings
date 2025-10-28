# frozen_string_literal: true
require "httparty"
require "nokogiri"
require "htmlentities"
require "json"
require "json-schema"
require "roo"
require "csv"

CVXCode = Struct.new("CVXCode", :short_description, :full_vaccine_name, :cvx_code, :notes, :status, :last_updated)
CPTCode = Struct.new("CPTCode", :cpt_code, :cpt_desc, :status, :comments, :vaccine_name, :cvx_code, :last_updated)
MVXCode = Struct.new("MVXCode", :cdc_product_name, :short_description, :cvx_code, :manufacturer, :mvx_code, :mvx_status, :product_name_status, :last_updated)
VaccineGroup = Struct.new("VaccineGroup", :short_description, :cvx_code, :status, :vaccine_group_name, :cvx_for_vaccine_group)

# Convert a map of XML values that look like <Name>AttributeName</name><value>AttributeValue</value> into a standard Ruby map
def parse_name_value_into_map(name_value_xml)
  result = {}
  cur_name = nil
  name_value_xml.children.each do |item|
    cur_name = item.children.first.to_s.strip if item.name.downcase == "name"
    result[cur_name] = item.children.first.to_s.strip if item.name.downcase == "value"
  end
  result
end

# Convert a standard Ruby map into a struct (or any object with direct key access)
def parsed_map_to_struct(parsed_data, incoming_struct)
  parsed_data.each do |k, v|
    converted_key = k.strip
      .tr(' ', '_')
      .gsub('CVXCode', 'cvx_code')
      .gsub('LastUpdated', 'last_updated')
      .gsub('ShortDescription', 'short_description')
      .downcase
      .to_sym
    incoming_struct[converted_key] = v
  end
  return incoming_struct
end

def map_to_cvx_code(parsed_data)
  res = CVXCode.new
  parsed_map_to_struct(parsed_data, res)
  return res
end

def map_to_cpt_code(parsed_data)
  res = CPTCode.new
  parsed_map_to_struct(parsed_data, res)
  return res
end

def map_to_mvx_code(parsed_data)
  res = MVXCode.new
  parsed_map_to_struct(parsed_data, res)
  return res
end

def map_to_vaccine_group(parsed_data)
  res = VaccineGroup.new
  parsed_map_to_struct(parsed_data, res)
  return res
end

desc "Load all of the appropriate CVX mappings from CDC-provided XML files."
task :load_cdc_mapping do
  cvx_to_all = {}

  # This expects the xml format from multiple CDC data sets.
  # <CVXCodes>
  #   <CVXInfo>
  #     <Name>Short Description</Name>
  #     <Value>Adenovirus types 4 and 7</Value>
  #     <Name>Full Vaccine name</Name>
  #     <Value>Adenovirus, type 4 and type 7, live, oral</Value>
  #     <Name>CVX Code</Name>
  #     <Value>143 </Value>
  #     <Name>Notes</Name>
  #     <Value>This vaccine is administered as 2 tablets.</Value>
  #     <Name>Status</Name>
  #     <Value>Active</Value>
  #     <Name>Last Updated</Name>
  #     <Value>3/20/2011</Value>
  #   </CVXInfo>
  # ...
  cvx_data_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=cvx")
  cvx_data_xml = cvx_data_xml_response.body
  cvx_data = Nokogiri::XML(cvx_data_xml)
  cvx = cvx_data.xpath("CVXCodes").xpath("CVXInfo")
  cvx.each do |node|
    parsed = parse_name_value_into_map(node)
    cvx_struct = map_to_cvx_code(parsed)
    cvx_to_all[cvx_struct.cvx_code] = { cvx: cvx_struct }
  end

  # Not all vaccines will have CPT or MVX codes. Groups are also not guaranteed.

  # <CPTCodes>
  #   <CPTInfo>
  # 	  <Name>CPT Code</Name>
  # 	  <Value>91300 </Value>
  # 	  <Name>CPT Desc</Name>
  # 	  <Value>
  # 	  Severe acute respiratory syndrome coronavirus 2 (SARS-CoV-2) (coronavirus disease [COVID-19]) vaccine, ...
  # 	  </Value>
  # 	  <Name>Status</Name>
  # 	  <Value/>
  # 	  <Name>Comments</Name>
  # 	  <Value>COVID-19 Vaccine - EUA authorization 12/11/2020</Value>
  # 	  <Name>Vaccine Name</Name>
  # 	  <Value>COVID-19, mRNA, LNP-S, PF, 30 mcg/0.3 mL dose</Value>
  # 	  <Name>CVX Code</Name>
  # 	  <Value>208 </Value>
  # 	  <Name>LastUpdated</Name>
  # 	  <Value>9/10/2021</Value>
  #   </CPTInfo>
  # ..
  # This list *includes inactive CPT codes as well*. The CVX code *may not map to an inactive CPT*!
  cpt_to_cvx_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=cpt")
  cpt_to_cvx_xml = cpt_to_cvx_xml_response.body
  cpt_to_cvx = Nokogiri::XML(cpt_to_cvx_xml)
  cpt_info = cpt_to_cvx.xpath("CPTCodes").xpath("CPTInfo")
  cpt_info.each do |node|
    parsed = parse_name_value_into_map(node)
    cpt_struct = map_to_cpt_code(parsed)
    cvx_code = cpt_struct.cvx_code
    
    if cvx_to_all.key? cvx_code
      # There's USUALLY only 1 CVX/CPT combo, but some times there are multiple options.
      unless cvx_to_all[cvx_code].key? :cpt
        cvx_to_all[cvx_code][:cpt] = []
      end

      if cvx_to_all[cvx_code][:cpt].length >= 1
        # puts "CPT code already existed for CVX code #{cvx_code}; existing code #{cvx_to_all[cvx_code][:cpt]['CPT Code']}, new code #{parsed['CPT Code']}"
        # puts "  Old\n  #{cvx_to_all[cvx_code][:cpt]}"
        # puts "  New\n  #{parsed}"
      end
      cvx_to_all[cvx_code][:cpt].append(cpt_struct)
    else
      puts "CPT didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { cpt: [cpt_struct] }
    end
  end

  # <productnames>
  # 	<prodInfo>
  # 		<Name>CDC Product Name</Name>
  # 		<Value>ACAM2000</Value>
  # 		<Name>Short Description</Name>
  # 		<Value>vaccinia (smallpox)</Value>
  # 		<Name>CVXCode</Name>
  # 		<Value>75 </Value>
  # 		<Name>Manufacturer</Name>
  # 		<Value>Acambis, Inc</Value>
  # 		<Name>MVX Code</Name>
  # 		<Value>ACA </Value>
  # 		<Name>MVX Status</Name>
  # 		<Value>Inactive</Value>
  # 		<Name>Product name Status</Name>
  # 		<Value>Inactive</Value>
  # 		<Name>Last Updated</Name>
  # 		<Value>5/28/2010</Value>
  # 	</prodInfo>
  # ...
  # A single CVX/CPT code can have multiple MVX codes
  mvx_to_cvx_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=tradename")
  mvx_to_cvx_xml = mvx_to_cvx_xml_response.body
  mvx_to_cvx = Nokogiri::XML(mvx_to_cvx_xml)
  prod_info = mvx_to_cvx.xpath("productnames").xpath("prodInfo")
  prod_info.each do |node|
    parsed = parse_name_value_into_map(node)
    mvx_struct = map_to_mvx_code(parsed)
    cvx_code = mvx_struct.cvx_code
    if cvx_to_all.key? cvx_code
      unless cvx_to_all[cvx_code].key? :mvx
        cvx_to_all[cvx_code][:mvx] = []
      end
      cvx_to_all[cvx_code][:mvx].append(mvx_struct)
    else
      puts "MVX didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { mvx: [mvx_struct] }
    end
  end

  # <VGCodes>
  #   <CVXVGInfo>
  #     <Name>ShortDescription</Name>
  #     <Value>DTP</Value>
  #     <Name>CVXCode</Name>
  #     <Value>01 </Value>
  #     <Name>Status</Name>
  #     <Value>Inactive</Value>
  #     <Name>Vaccine Group Name</Name>
  #     <Value>DTAP</Value>
  #     <Name>CVX for Vaccine Group</Name>
  #     <Value>107</Value>
  #   </CVXVGInfo>
  # ...
  # Not all CVX codes will have group information
  cvx_grouping_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=vax2vg")
  cvx_grouping_xml = cvx_grouping_xml_response.body
  cvx_grouping = Nokogiri::XML(cvx_grouping_xml)
  cvx_vg_info = cvx_grouping.xpath("VGCodes").xpath("CVXVGInfo")
  cvx_vg_info.each do |node|
    parsed = parse_name_value_into_map(node)
    cvx_vg_struct = map_to_vaccine_group(parsed)
    cvx_code = cvx_vg_struct.cvx_code
    if cvx_to_all.key? cvx_code
      if cvx_to_all[cvx_code].key? :groups
        cvx_to_all[cvx_code][:groups].append(cvx_vg_struct)
      else
        cvx_to_all[cvx_code][:groups] = [cvx_vg_struct]
      end
    else
      puts "CVX didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { groups: cvx_vg_struct }
    end
  end

  extra_cvx_data(cvx_to_all).each do |cvx_code, cvx_data|
    raise "Custom CVX code #{cvx_code} already exists in CDC data" if cvx_to_all.key?(cvx_code.to_s)

    cvx_to_all[cvx_code.to_s] = cvx_data
  end

  # Now produce a JSON file with all of the consolidated data we want
  json_result = { cvx: {}, cpt: {} }
  tn_overrides = trade_name_overrides
  # The entity encoder converts HTML escape chars like &amp; to their UTF-8 equivalent.
  entity_encoder = HTMLEntities.new

  extra_cpt_data.each do |cvx_code, cpt_structs|
    if !cvx_to_all.key?(cvx_code.to_s)
      raise StandardError, "Found a manually-defined CPT code WITHOUT an existing CVX code. CVX code given is #{cvx_code}."
    end
    cvx_to_all[cvx_code.to_s][:cpt] = cvx_to_all[cvx_code.to_s][:cpt] + cpt_structs
  end

  cvx_to_all.each do |k, v|
    # The key is a CVX code and the value is the group name.
    # There are some vaccines in multiple groups like a combo Hep A-Hep B vaccine.
    groups = {}
    if v.key? :groups
      v[:groups].each do |group|
        groups[group.cvx_for_vaccine_group] = {
          name: entity_encoder.decode(group.vaccine_group_name),
          active: (group.status == "Active")
        }
      end
      #group_name = v[:groups]["Vaccine Group Name"]
      #group_cvx = v[:groups]["CVX for Vaccine Group"]
    end
    groups = groups.sort_by { |cvx, _| cvx.to_i }.to_h
    # If there are no groups, just use the current vaccine info.
    if groups.nil? || groups.empty?
      groups[k] = {
        name: v[:cvx].short_description.upcase,
        active: true
      }
    end

    manufacturers = []
    if v.key? :mvx
      manufacturers = v[:mvx].map do |m|
        trade_name = m.cdc_product_name
        trade_name = tn_overrides[k] if tn_overrides.key? k
        last_updated = Date.strptime(m.last_updated, "%m/%d/%Y")
        {
          trade_name: entity_encoder.decode(trade_name),
          mvx_code: m.mvx_code,
          manufacturer: entity_encoder.decode(m.manufacturer),
          last_updated: last_updated.strftime("%Y-%m-%d"),
          product_name_status: m.product_name_status
        }
      end
    end
    manufacturers = manufacturers.sort_by { |mfn| mfn['mvx_code'] }

    cpt_codes = []
    if v.key?(:cpt)
      cpt_codes = v[:cpt].map do |cpt|
        {
          cpt_code: cpt.cpt_code,
          cpt_desc: cpt.cpt_desc
        }
      end
    end

    item = {
      cvx_code: k,
      name: entity_encoder.decode(v[:cvx].short_description),
      description: entity_encoder.decode(v[:cvx].short_description),
      status: v[:cvx].status,
      cpt_codes: cpt_codes,
      groups: groups,
    #group_name: entity_encoder.decode(group_name),
    #group_cvx: group_cvx,
      manufacturers: manufacturers
    }

    if json_result[:cvx].key? k
      raise "Found two of the same CVX keys; this should not happen: #{k}. Current item: #{json_results[:cvx][k]}. New item: #{item}."
    end

    json_result[:cvx][k] = item
    if v.key? :cpt
      cpt_codes.each do |cpt_code_item|
        cpt_code = cpt_code_item[:cpt_code]
        if json_result[:cpt].key? cpt_code
          puts "CPT key #{cpt_code} already exists.\n  Cur: #{json_result[:cpt][cpt_code]}\n  New: #{item}.\n\n"
        else
          json_result[:cpt][cpt_code] = item
        end
      end
    end
  end

  # Sort for stable changes between updates
  json_result[:cvx] = json_result[:cvx].sort.to_h
  json_result[:cvx].each do |k, v|
    json_result[:cvx][k][:cpt_codes] = v[:cpt_codes].sort_by { |i| i[:cpt_code] }
    json_result[:cvx][k][:manufacturers] = v[:manufacturers].sort_by { |i| "#{i[:trade_name]}#{i[:mvx_code]}" }
    json_result[:cvx][k][:groups] = v[:groups].sort_by { |i, _| i.to_i }.to_h
  end

  json_result[:cpt] = json_result[:cpt].sort.to_h
  json_result[:cpt].each do |k, v|
    json_result[:cpt][k][:cpt_codes] = v[:cpt_codes].sort_by { |i| i[:cpt_code] }
    json_result[:cpt][k][:manufacturers] = v[:manufacturers].sort_by { |i| "#{i[:trade_name]}#{i[:mvx_code]}" }
    json_result[:cpt][k][:groups] = v[:groups].sort_by { |i, _| i.to_i }.to_h
  end

  JSON::Validator.validate!('vaccine-code-mapping-schema.json', json_result)
  File.write("./vaccine-code-mapping.json", JSON.pretty_generate(json_result))
end

# From the 2014 USIIS implementation guide: "Note: Utah uses one valid custom vaccine code, CVX 943: HepB, 2 Dose (11-15
# yrs, Merck only). This code may be returned in VXR messages."
#
# We should coerce map this to the same values used for CVX code 43. 43 is the obvious choice since Merck is a
# manufacturer, it appears USIIS has simply prefixed it with "9", AND we can see in the ICE documentation that the
# Adolescent 2-Dose Series Rules exception references CVX 43 specifically with similar wording to the USIIS
# implementation guide: "If a.) CVX 43 (Hep B adult) is administered to a patient >= 11 years and < 16 years as dose 1
# and dose 2 AND b.) there is a minimum interval of 4 months - 4 days between dose 1 and dose 2, THEN the series is
# complete with 2 doses. If this rule is not met, default to the Hep B 3-dose Child/Adolescent Series."
def extra_cvx_data(existing)
  existing_hepb = existing['43']
  raise 'CVX code 43 does not exist in the existing data' if existing_hepb.nil?

  {
    # Duplicate all the information for code 43 and attach it to the new custom code 943.
    "943": existing_hepb
  }
end

def extra_cpt_data
  # CPTCode = Struct.new("CPTCode", :cpt_code, :cpt_desc, :status, :comments, :vaccine_name, :cvx_code, :last_updated)
  {
    # This was added for Minnesota, which is importing Diphtheria without a CVX code with an outdated/unused CPT code.
    "12": [
      CPTCode.new(cpt_code: "90719", cpt_desc: "Diphtheria toxoid, for intramuscular use", status: "Inactive", comments: "", vaccine_name: "Diphtheria", cvx_code: "12", last_updated: "07/01/2019" )
    ]
  }
end

def trade_name_overrides
  # These only really need to be defined for COVID-19 for now since they're rapidly changing.
  # The data comes from https://www.cdc.gov/vaccines/programs/iis/COVID-19-related-codes.html
  # It's not included as part of the regular download/processing since only excel files are available.
  # Note that only the US vaccines are here since the foreign vaccines
  # are correctly named in the regular CVX code set.
  {
    "207": "Moderna COVID-19 Vaccine",
    "208": "Pfizer-BioNTech COVID-19 Vaccine",
    "210": "AstraZeneca COVID-19 Vaccine",
    "211": "Novavax COVID-19 Vaccine",
    "212": "Janssen COVID-19 Vaccine",
    "213": "SARS-COV-2 (COVID-19) vaccine, UNSPECIFIED",
    "217": "Pfizer-BioNTech COVID-19 Vaccine",
    "218": "Pfizer-BioNTech COVID-19 Vaccine",
    "219": "Pfizer-BioNTech COVID-19 Vaccine"
  }
end


desc "Compare MIIC mapping to the CDC mapping of already-downloaded files."
task :compare_mapping do
  miic_map_path = "./miic-mapping.csv"
  miic_data = File.read(miic_map_path)
  miic_csv = CSV.parse(miic_data, headers: true)

  cdc_map_path = "./vaccine-code-mapping.json"
  cdc_data = File.read(cdc_map_path)
  cdc_json = JSON.parse(cdc_data)

  cdc_codes = cdc_json["cvx"].keys.sort_by { |c| c.to_i }
  miic_codes = miic_csv["CVX_CODE"].filter { |c| !c.nil? && c != '' }.map { |c| c.to_s.strip }.sort

  puts "Vaccine Group Name Mismatches - show on the main 'Records' page in Docket"
  puts "CVX_Group_Code,CDC_Name,MIIC_Name"
  cdc_groups = cdc_json["cvx"].map { |k, v| v["groups"] }.flatten.map { |k| { cvx: k.first[0], name: k.first[1]["name"] || '' } }.uniq.sort_by { |g| g[:cvx].to_i }
  miic_groups = miic_csv.map { |row| { cvx: row["CVX_CODE"] || '', name: row["VACCINE_GROUP"] } }.uniq.sort_by { |g| g[:cvx] }
  cdc_groups.each do |cdc_group|
    miic_group = miic_groups.find { |row| row[:cvx] == cdc_group[:cvx] }
    if miic_group.nil?
      next
    end

    # Compare the two items
    if cdc_group[:name] != miic_group[:name]
      puts "#{cdc_group[:cvx]},\"#{cdc_group[:name]}\",\"#{miic_group[:name]}\""
    end
  end

  puts ""
  puts "Exact Vaccine Name Mismatches - show on the PDF and in the vaccine details screen in Docket"
  puts "CVX_Code,CDC_Name,MIIC_Name"
  cdc_codes.each do |cdc_cvx|
    cdc_item = cdc_json["cvx"][cdc_cvx]
    miic_item = miic_csv.find { |row| row["CVX_CODE"] == cdc_cvx }
    if miic_item.nil?
      next
    end

    # Compare the two items
    if cdc_item["name"] != miic_item["VACCINE"]
      puts "#{cdc_cvx},\"#{cdc_item["name"]}\",\"#{miic_item["VACCINE"]}\""
    end
  end


  puts ""
  only_cdc = cdc_codes - miic_codes
  puts "CVX codes only in the CDC data set: #{only_cdc}"
  only_miic = miic_codes - cdc_codes
  puts "CVX codes only in the MIIC data set: #{only_miic}"
end

desc "Download and convert the Excel file for MIIC"
task :load_miic_mapping do
  # This was removed from the GitHub workflow on 2025-05-07.
  # MIIC seems to have removed the data file, so this will fail indefinitely.
  # There is an email out to Aaron, Miriam, and Sydney from MN to ask what happened.

  def trade_name_overrides
    # These only really need to be defined for COVID-19 for now since they're rapidly changing.
    # The data comes from https://www.cdc.gov/vaccines/programs/iis/COVID-19-related-codes.html
    # It's not included as part of the regular download/processing since only excel files are available.
    # Note that only the US vaccines are here since the foreign vaccines
    # are correctly named in the regular CVX code set.
    {
      "207": "Moderna COVID-19 Vaccine",
      "208": "Pfizer-BioNTech COVID-19 Vaccine",
      "210": "AstraZeneca COVID-19 Vaccine",
      "211": "Novavax COVID-19 Vaccine",
      "212": "Janssen COVID-19 Vaccine",
      "213": "SARS-COV-2 (COVID-19) vaccine, UNSPECIFIED",
      "217": "Pfizer-BioNTech COVID-19 Vaccine",
      "218": "Pfizer-BioNTech COVID-19 Vaccine",
      "219": "Pfizer-BioNTech COVID-19 Vaccine"
    }
  end
  miic_map_path = "./miic-mapping.csv"
  url = "https://www.health.state.mn.us/people/immunize/miic/data/vaxcodes.xlsx"
  miic_xls = HTTParty.get(url, headers: {
    "User-Agent" => "Mozilla/5.0 (X11; Linux x86_64) Docket/1.0", # MIIC gives back a 403 if a user agent is not populated
  })
  miic_xls_data = StringIO.new miic_xls.body
  xls_file = Roo::Excelx.new(miic_xls_data)
  File.write(miic_map_path, xls_file.to_csv)
  # This is kind of dumb, but it's the quickest way to get this accomplished
  csv_data = File.read(miic_map_path)
  lines = csv_data.split("\n")
  header = lines[0].tr(" ", "_")
  csv_with_replaced_header = ([header] + lines[1..]).join("\n")
  File.write(miic_map_path, csv_with_replaced_header)
end
