#!/usr/bin/perl
use strict; use warnings;
use LWP::UserAgent;
use JSON;

my $API_KEY = 'YOUR_API_KEY';
my $BASE    = 'https://api.ip-block.com/v1';

# Helper: returns 1 if the API is reachable
sub api_reachable {
    my $ua = LWP::UserAgent->new(timeout => 0.5);
    my $res = $ua->get("$BASE/ping");
    return 0 unless $res->is_success;
    my $data = decode_json($res->content);
    return ($data->{status} eq 'ok') ? 1 : 0;
}

# Check visitor IP; returns 1 if allowed
sub visitor_allowed {
    my ($ip) = @_;
    return 1 unless api_reachable(); # fail open

    my $ua  = LWP::UserAgent->new(timeout => 2);
    my $res = $ua->post(
        "$BASE/check",
        'Content-Type' => 'application/json',
        Content => encode_json({
            api_key    => $API_KEY,
            ip         => $ip,
            site_id    => 'ABCDEFGHIJKL',
            user_agent => $ENV{HTTP_USER_AGENT} // '',  # optional
            referrer   => $ENV{HTTP_REFERER}    // '',  # optional
        })
    );
    my $data = decode_json($res->content);
    return ($data->{action} eq 'block') ? 0 : 1;
}

# CGI usage
my $ip = $ENV{REMOTE_ADDR};
unless (visitor_allowed($ip)) {
    print "Status: 302 Found\r\nLocation: https://www.ip-block.com/blocked.php\r\n\r\n";
    exit;
}