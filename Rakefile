# frozen_string_literal: true
require "httparty"
require "nokogiri"
require "htmlentities"
require "json"
require "json-schema"
require "roo"

def parse_name_value_into_map(name_value_xml)
  result = {}
  cur_name = nil
  name_value_xml.children.each do |item|
    cur_name = item.children.first.to_s.strip if item.name.downcase == "name"
    result[cur_name] = item.children.first.to_s.strip if item.name.downcase == "value"
  end
  result
end

desc "Download and convert the Excel file for MIIC"
task :load_miic_mapping do
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

desc "Load all of the appropriate CVX mappings from CDC-provided XML files."
task :load_cdc_mapping do
  cvx_to_all = {}

  # This expects the xml format from multiple CDC data sets.
  # <CVXCodes>
  #   <CVXInfo>
  #     <ShortDescription>Adenovirus types 4 and 7</ShortDescription>
  #     <FullVaccinename>Adenovirus, type 4 and type 7, live, oral</FullVaccinename>
  #     <CVXCode>143 </CVXCode>
  #     <Notes>This vaccine is administered as 2 tablets.</Notes>
  #     <Status>Active</Status>
  #     <LastUpdated>3/20/2011</LastUpdated>
  #   </CVXInfo>
  # ...
  cvx_data_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=cvx")
  cvx_data_xml = cvx_data_xml_response.body
  cvx_data = Nokogiri::XML(cvx_data_xml)
  cvx = cvx_data.xpath("CVXCodes").xpath("CVXInfo")
  cvx.each do |node|
    parsed = parse_name_value_into_map(node)
    cvx_to_all[parsed["CVX Code"]] = { cvx: parsed }
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
  cpt_to_cvx_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=cpt")
  cpt_to_cvx_xml = cpt_to_cvx_xml_response.body
  cpt_to_cvx = Nokogiri::XML(cpt_to_cvx_xml)
  cpt_info = cpt_to_cvx.xpath("CPTCodes").xpath("CPTInfo")
  cpt_info.each do |node|
    parsed = parse_name_value_into_map(node)
    cvx_code = parsed["CVX Code"]
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
      cvx_to_all[cvx_code][:cpt].append(parsed)
    else
      puts "CPT didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { cpt: [parsed] }
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
    cvx_code = parsed["CVXCode"]
    if cvx_to_all.key? cvx_code
      unless cvx_to_all[cvx_code].key? :mvx
        cvx_to_all[cvx_code][:mvx] = []
      end
      cvx_to_all[cvx_code][:mvx].append(parsed)
    else
      puts "MVX didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { mvx: [parsed] }
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
  #     </CVXVGInfo>
  # ...
  # Not all CVX codes will have group information
  cvx_grouping_xml_response = HTTParty.get("https://www2a.cdc.gov/vaccines/iis/iisstandards/XML.asp?rpt=vax2vg")
  cvx_grouping_xml = cvx_grouping_xml_response.body
  cvx_grouping = Nokogiri::XML(cvx_grouping_xml)
  cvx_vg_info = cvx_grouping.xpath("VGCodes").xpath("CVXVGInfo")
  cvx_vg_info.each do |node|
    parsed = parse_name_value_into_map(node)
    cvx_code = parsed["CVXCode"]
    if cvx_to_all.key? cvx_code
      if cvx_to_all[cvx_code].key? :groups
        #puts "Already have CVX group for CVX code #{cvx_code}!"
        cvx_to_all[cvx_code][:groups].append(parsed)
      else
        cvx_to_all[cvx_code][:groups] = [parsed]
      end
    else
      puts "CVX didn't exist for #{cvx_code}"
      cvx_to_all[cvx_code] = { groups: parsed }
    end
  end

  # Now produce a JSON file with all of the consolidated data we want
  json_result = { cvx: {}, cpt: {} }
  tn_overrides = trade_name_overrides
  # The entity encoder converts HTML escape chars like &amp; to their UTF-8 equivalent.
  entity_encoder = HTMLEntities.new
  cvx_to_all.each do |k, v|
    # The key is a CVX code and the value is the group name.
    # There are some vaccines in multiple groups like a combo Hep A-Hep B vaccine.
    groups = {}
    if v.key? :groups
      v[:groups].each do |group|
        groups[group["CVX for Vaccine Group"]] = {
          name: entity_encoder.decode(group["Vaccine Group Name"]),
          active: (group["Status"] == "Active")
        }
      end
      #group_name = v[:groups]["Vaccine Group Name"]
      #group_cvx = v[:groups]["CVX for Vaccine Group"]
    end
    groups = groups.sort_by { |cvx, _| cvx.to_i }.to_h
    # If there are no groups, just use the current vaccine info.
    if groups.nil? || groups.empty?
      groups[k] = {
        name: v[:cvx]["Short Description"].upcase,
        active: true
      }
    end

    manufacturers = []
    if v.key? :mvx
      manufacturers = v[:mvx].map do |m|
        trade_name = m["CDC Product Name"]
        trade_name = tn_overrides[k] if tn_overrides.key? k
        last_updated = Date.strptime(m["Last Updated"], "%m/%d/%Y")
        {
          trade_name: entity_encoder.decode(trade_name),
          mvx_code: m["MVX Code"],
          manufacturer: entity_encoder.decode(m["Manufacturer"]),
          last_updated: last_updated.strftime("%Y-%m-%d"),
          product_name_status: m["Product name Status"]
        }
      end
    end
    manufacturers = manufacturers.sort_by { |mfn| mfn["mvx_code"] }

    cpt_codes = []
    if v.key? :cpt
      cpt_codes = v[:cpt].map do |cpt|
        {
          cpt_code: cpt["CPT Code"],
          cpt_desc: cpt["CPT Desc"]
        }
      end
    end

    item = {
      cvx_code: k,
      name: entity_encoder.decode(v[:cvx]["Short Description"]),
      description: entity_encoder.decode(v[:cvx]["Short Description"]),
      status: v[:cvx]["Status"],
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
          puts "CPT key #{cpt_code} already exists.\n  Cur: #{json_result[:cpt][cpt_code]}\n  New: #{item}."
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
