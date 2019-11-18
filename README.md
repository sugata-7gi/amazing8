# amazing8
#!/usr/bin/env ruby
# -*- coding: utf-8 -*-

require 'rubygems'
require 'adsense-info'
require 'mechanize'
require 'rexml/document'
require 'date'

AMAZON_EMAIL = "someone@mailserver.com"
AMAZON_PASSWORD = "password"

GOOGLE_ACCOUNT = "someone"
GOOGLE_PASSWORD = "password"

FILE_PATH = "/Users/someone/affiliate-info.txt"

#----------------------------------------------------------

class AmazonReport < Object

  ROOT = "https://affiliate.amazon.co.jp/gp/associates"
  LOGIN = "#{ROOT}/login/login.html"
  REPORT = "#{ROOT}/network/reports/report.html"

  USER_AGENT = "Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_6_3; ja-jp) " +
    "AppleWebKit/533.16 (KHTML, like Gecko) Version/5.0 Safari/533.16"

  def initialize(email, password)
    @agent = Mechanize::new
    # User-Agentの偽装は必須
    @agent.user_agent = USER_AGENT
    login_form = @agent.get(LOGIN).forms.first
    login_form.field_with(:name => 'email').value = email
    login_form.field_with(:name => 'password').value = password
    @agent.submit(login_form)
    self.initialize_default_date
  end

  def initialize_default_date
    start_day = Date.new(Date.today.year, Date.today.mon, 1)
    today = Date.today
    @start_year = @end_year = today.year
    @start_month = @end_month = today.month
    @start_day = start_day.day
    @end_day = today.day
  end

  def build_params(params)
    return params.map{ |k, v| "#{k}=#{v}" }.join('&')
  end

  def params(start_year, start_month, start_day,
             end_year, end_month, end_day)
    # なぜか月は0ベースで一つ前の数字を指定
    params = {
      '__mk_ja_JP' => 'カタカナ',
      'tag' => '',
      'reportType' => 'earningsReport',
      'program' => 'all',
      'periodType' => 'exact',
      'preSelectedPeriod' => 'monthToDate',
      'startYear' => start_year.to_s,
      'startMonth' => (start_month - 1).to_s,
      'startDay' => start_day.to_s,
      'endYear' => end_year.to_s,
      'endMonth' => (end_month - 1).to_s,
      'endDay' => end_day.to_s,
      'submit.display.x' => '1',
      'submit.display.y' => '1',
      'submit.download_XML' => 'レポートのダウンロード（XML形式）' 
    }
    return self.build_params(params)
  end

  def earnings(start_year=@start_year, start_month=@start_month, start_day=@start_day,
             end_year=@end_year, end_month=@end_month, end_day=@end_day)
    params = self.params(start_year, start_month, start_day,
                         end_year, end_month, end_day)
    page = @agent.get("#{REPORT}?#{params}")
    doc = REXML::Document.new(page.body)
    return doc.elements['/Data/ReferralFeeTotals/Earnings'].text
  end

end

adsense = Adsense::Info.new(GOOGLE_ACCOUNT, GOOGLE_PASSWORD)

begin
  # 初回はなぜか例外発生
  adsense.today_so_far
rescue
  #
end

# 今日のAdSenseの収入
adsense = adsense.today_so_far.gsub("&#65509;", "")

ar = AmazonReport.new(AMAZON_EMAIL, AMAZON_PASSWORD)
# 当月の紹介料
amazon = ar.earnings

# 期間を指定する場合
# puts ar.earnings(2011, 1, 1, 2011, 1, 31)

File.open(FILE_PATH, 'w') {|f|
  f.puts "¥#{adsense} / ¥#{amazon}"
}
